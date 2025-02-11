<script>
	import { DataTable } from "carbon-components-svelte";
	import JSZip from "jszip";
	import { uploadImage } from "$lib/supabase";
	import { onMount } from "svelte";
	import QrScanner from "qr-scanner";
	import toast from "svelte-french-toast";
	import { handleError } from "$lib/handleError";
	import Button from "$lib/components/Button.svelte";

	// When rendering to canvas, the scaling we use.
	const PDF_SCALE = 2.0;
	// Location in pts of the top right qr code
	const TEST_ID_PAGE_BOX = { left: 465, top: 14, width: 70, height: 70 };
	// Location in pts of the sticker qr code (test taker ID)
	const FRONT_ID_BOX = { left: 335, top: 130, width: 57.6, height: 57.6 };
	// How many pts to expand the bounding boxes when looking to recognize the qr codes.
	const QR_SEARCH_PADDING = 50;

	let files = [];
	let pngs_to_upload = new Map();

	$: if (files) {
		(async () => {
			try {
				await handleFileUpload();
			} catch (error) {
				handleError(error);
				toast.error(error.message);
			}
		})();
	}

	function initializePDFJS() {
		let pdf_js_loaded = new Promise(async (resolve, reject) => {
			let counter = 0;
			function is_pdf_js_loaded() {
				if (typeof pdfjsLib !== "undefined") {
					resolve();
				} else {
					counter += 1;
					// 10 seconds later, if pdf js has not loaded, throw an error.
					if (counter > 100) {
						reject("PDF.js did not load in 10 seconds.");
					}
					// Every .1 seconds
					setTimeout(is_pdf_js_loaded, 100);
				}
			}
			is_pdf_js_loaded();
		});

		pdf_js_loaded.then(() => {
			console.log("pdf.js loaded");
			pdfjsLib.GlobalWorkerOptions.workerSrc =
				"https://cdnjs.cloudflare.com/ajax/libs/pdf.js/4.0.379/pdf.worker.min.mjs";
		});
	}

	onMount(initializePDFJS);

	async function handleFileUpload() {
		const filesToProcess = [];
		for (const file of files) {
			if (
				file.type === "application/zip" ||
				file.type === "application/x-zip-compressed"
			) {
				filesToProcess.push(...(await unzipFile(file)));
			} else {
				filesToProcess.push(file);
			}
		}

		console.log(filesToProcess);
		const file_pngs = await Promise.all(
			filesToProcess.map(async (file) => [
				file.name,
				await convertToPngs(await file.arrayBuffer()),
			])
		);
		const expand_box = (bounding_box) => {
			const b = bounding_box;
			return {
				x: (b.left - QR_SEARCH_PADDING) * PDF_SCALE,
				y: (b.top - QR_SEARCH_PADDING) * PDF_SCALE,
				width: (b.width + 2 * QR_SEARCH_PADDING) * PDF_SCALE,
				height: (b.height + 2 * QR_SEARCH_PADDING) * PDF_SCALE,
			};
		};

		const scan_box = (png, bounding_box) => {
			return new Promise((resolve, reject) =>
				QrScanner.scanImage(png, {
					scanRegion: expand_box(bounding_box),
					returnDetailedScanResult: true,
				})
					.then(resolve)
					.catch((e) => reject(e || "No QR code found"))
			);
		};
		const scan_boxes = async (png, test_id_page_box, front_id_box) => {
			const { data: test_id_page } = await scan_box(png, test_id_page_box);
			const { data: front_id } = await scan_box(png, front_id_box);
			return [test_id_page, front_id, png];
		};

		for (const [file_name, pngs] of file_pngs) {
			for (const png of pngs) {
				// Try to get either the top right or the bottom left qr boxes.
				let test_id_page, front_id, matched_png;
				try {
					[test_id_page, front_id, matched_png] = await Promise.any([
						scan_boxes(png[0], TEST_ID_PAGE_BOX, FRONT_ID_BOX),
						scan_boxes(png[1], TEST_ID_PAGE_BOX, FRONT_ID_BOX),
					]);
				} catch {
					// Skip if no QR codes are found.
					continue;
				}

				// Assert that the test id matches a certain pattern.
				if (!test_id_page.match(/T\d+P\d+/)) {
					throw "Expected test and page id in T\\d+P\\d+ format.";
				}
				const [start, end] = test_id_page.split("P");
				const test_id = start.substr(1);
				// Convert from 1 indexed to 0 indexed.
				const page = parseInt(end) - 1;
				const identifier = test_id + page + front_id;

				// Handle already loaded conflicts.
				if (pngs_to_upload.has(identifier)) {
					pngs_to_upload = pngs_to_upload;
					throw new Error(
						`Found duplicated identifier: T${test_id}P${page} ${front_id} in ${file_name}. \
						Conflicts with ${pngs_to_upload.get(identifier).file_name}`
					);
				}
				pngs_to_upload.set(identifier, {
					file_name,
					matched_png,
					blob_url: URL.createObjectURL(matched_png),
					test_id,
					page: page.toString(),
					front_id,
				});
			}
		}
		// Trigger svelte to run listeners.
		pngs_to_upload = pngs_to_upload;
	}

	async function unzipFile(zipFile) {
		const zip = new JSZip();
		const zipData = await zip.loadAsync(zipFile);

		const fileList = [];
		zipData.forEach(async (relativePath, zipEntry) =>
			fileList.push([relativePath, zipEntry])
		);

		// Mac Finder-generated zip files contain extra __MACOSX files that are redundant.
		const extracted_files = fileList
			.filter(
				([_, zipEntry]) =>
					!zipEntry.dir &&
					!zipEntry.name.startsWith("__MACOSX") &&
					zipEntry.name.endsWith(".pdf")
			)
			.map(([_relativePath, zipEntry]) => {
				return new Promise(async (resolve, _reject) => {
					const extractedFileData = await zipEntry.async("arraybuffer");
					const extractedFile = new File([extractedFileData], zipEntry.name);
					resolve(extractedFile);
				});
			});

		return Promise.all(extracted_files);
	}

	// Returns the rendered PDF file as a blob and as rotated.
	function convertToPngs(file) {
		const loadingTask = pdfjsLib.getDocument({ data: new Uint8Array(file) });

		// Render a single pdf page onto a canvas with pdf.js
		const convert_page_to_png = async (page, canvasdiv) => {
			const viewport = page.getViewport({ scale: PDF_SCALE });

			const canvas = document.createElement("canvas");
			canvasdiv.appendChild(canvas);

			// Prepare canvas using PDF page dimensions
			const context = canvas.getContext("2d");
			canvas.height = viewport.height;
			canvas.width = viewport.width;

			// Render PDF page into canvas context
			const renderContext = { canvasContext: context, viewport: viewport };

			const renderTask = page.render(renderContext);
			const [blob, rotated_blob] = await renderTask.promise.then(async () => {
				const unrotated = await new Promise((resolve) => {
					canvas.toBlob(resolve);
				});
				context.translate(canvas.width / 2, canvas.height / 2);
				context.rotate(Math.PI);
				context.drawImage(
					context.canvas,
					0,
					0,
					canvas.width,
					canvas.height,
					-canvas.width / 2,
					-canvas.height / 2,
					canvas.width,
					canvas.height
				);

				const rotated = await new Promise((resolve) => {
					canvas.toBlob(resolve);
				});
				return [unrotated, rotated];
			});

			canvas.remove();
			return [blob, rotated_blob];
		};

		return loadingTask.promise.then(
			async (pdf) => {
				const canvas_div = document.getElementById("canvas");
				const totalPages = pdf.numPages;
				console.log(`processing ${totalPages} pages`);
				let data = [];

				for (let pageNumber = 1; pageNumber <= totalPages; pageNumber++) {
					let page = await pdf.getPage(pageNumber);
					data.push(await convert_page_to_png(page, canvas_div));
				}
				return data;
			},
			function (reason) {
				// PDF loading error
				console.error(reason);
			}
		);
	}

	async function upload_scans() {
		try {
			for (const [_, png] of pngs_to_upload.entries()) {
				await uploadImage(png.matched_png, png.test_id, png.page, png.front_id);
			}
			pngs_to_upload.clear();
			pngs_to_upload = pngs_to_upload;
		} catch (error) {
			handleError(error);
			toast.error(error.message);
		}
	}
</script>

<head>
	<script
		type="module"
		src="https://cdnjs.cloudflare.com/ajax/libs/pdf.js/4.0.379/pdf.min.mjs"
	></script>
</head>

<h2>Upload Scans</h2>

<br />

<label>
	<input type="file" multiple bind:files accept=".pdf,.zip" />
</label>

<br />
<br />

<Button title="Upload Scans" action={upload_scans} />

<br />
<br />

<DataTable
	sortable
	size="compact"
	expandable
	headers={[
		{ key: "file_name", value: "File Name" },
		{ key: "test_id", value: "Test ID" },
		{ key: "page", value: "Page Index" },
		{ key: "front_id", value: "Taker ID" },
	]}
	rows={[
		...pngs_to_upload.entries().map(([k, v]) => {
			return { id: k, ...v };
		}),
	]}
>
	<svelte:fragment slot="cell" let:cell>
		<div>
			<div style="overflow: hidden;">
				{cell.value == null || cell.value == "" ? "None" : cell.value}
			</div>
		</div>
	</svelte:fragment>
	<svelte:fragment slot="expanded-row" let:row>
		<div class="flex">
			<div
				style="border: 2px solid black;width: 50%;margin: 10px;padding: 10px;"
			>
				<img src={row.blob_url} class="scan-preview" />
			</div>
		</div>
	</svelte:fragment>
</DataTable>

<div id="canvas" style="display: none;" />

<style>
	img.scan-preview {
		width: 100%;
	}
</style>
