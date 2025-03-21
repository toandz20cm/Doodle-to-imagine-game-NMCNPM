<script lang="ts">
	import { onMount, tick } from 'svelte';
	import ShareWithCommunity from "$lib/ShareWithCommunity.svelte";
  
	let promptTxt = '';
	let strength = '0.85';
	let imagesReturned = '0';
	let isLoading = false;
	let isUploading = false;
	let isOutputControlAdded = false;
	let isSuccessfulGeneration = false;
	let drawingBoard: any;
	let canvas: HTMLCanvasElement;
	let ctx: CanvasRenderingContext2D | null;
	let noiseTs: DOMHighResTimeStamp;
	let imageTs: DOMHighResTimeStamp;
	let drawNextImage: () => void;
	let interval: ReturnType<typeof setInterval>;
	let canvasSize = 400;
	let canvasContainerEl: HTMLDivElement;
	let fileInput: HTMLInputElement;
	let sketchEl: HTMLCanvasElement;
	let isShowSketch = false;
	let outputImgs: CanvasImageSource[] = [];
	let outputFiles: { sketch: File; generations: File[] };
  
	const animImageDuration = 500 as const;
	const animNoiseDuration = 3000 as const;
  
	async function drawNoise() {
	  if (!ctx) return;
  
	  const imageData = ctx.createImageData(canvas.width, canvas.height);
	  const pix = imageData.data;
  
	  for (let i = 0, n = pix.length; i < n; i += 4) {
		const c = 7;
		pix[i] = 40 * Math.random() * c;
		pix[i + 1] = 40 * Math.random() * c;
		pix[i + 2] = 40 * Math.random() * c;
		pix[i + 3] = 10;
	  }
  
	  const bitmap = await createImageBitmap(imageData);
  
	  const duration = performance.now() - noiseTs;
	  ctx.globalAlpha = Math.min(duration, animNoiseDuration) / animNoiseDuration;
	  ctx.drawImage(bitmap, 0, 0, canvasSize, canvasSize);
  
	  if (isLoading) window.requestAnimationFrame(drawNoise);
	}
  
	function drawImage(image: CanvasImageSource) {
	  if (!ctx) return;
  
	  const duration = performance.now() - imageTs;
	  ctx.globalAlpha = Math.min(duration, animImageDuration) / animImageDuration;
	  ctx.drawImage(image, 0, 0, canvasSize, canvasSize);
  
	  if (duration < animImageDuration) window.requestAnimationFrame(() => drawImage(image));
	}
  
	async function getCanvasSnapshot(canvas: HTMLCanvasElement): Promise<{ imgFile: File; imgBitmap: ImageBitmap }> {
	  const canvasDataUrl = canvas.toDataURL('png');
	  const res = await fetch(canvasDataUrl);
	  const blob = await res.blob();
	  const imgFile = new File([blob], 'canvas shot.png', { type: 'image/png' });
	  const imgData = canvas.getContext('2d')!.getImageData(0, 0, canvasSize, canvasSize);
	  const imgBitmap = await createImageBitmap(imgData);
	  return { imgFile, imgBitmap };
	}
  
	async function submitRequest() {
	  if (!promptTxt) return alert('');
	  if (!canvas || !ctx) return;
  
	  if (interval) clearInterval(interval);
	  isLoading = true;
	  isShowSketch = false;
	  isSuccessfulGeneration = false;
	  copySketch();
  
	  noiseTs = performance.now();
	  drawNoise();
  
	  const { imgFile: sketch, imgBitmap: initialSketchBitmap } = await getCanvasSnapshot(canvas);
	  const form = new FormData();
	  form.append('prompt', promptTxt);
	  form.append('strength', strength);
	  form.append('image', sketch);
  
	  try {
		const response = await fetch('https://sdb.pcuenca.net/i2i', { method: 'POST', body: form });
		const json = JSON.parse(await response.text());
		const { images: imagesBase64Strs } = json;
  
		imagesReturned = imagesBase64Strs.length.toString();
  
		if (!imagesBase64Strs.length) return alert('');
  
		outputImgs = (await Promise.all(
		  imagesBase64Strs.map(async (imgBase64Str) => {
			const imgEl = new Image();
			imgEl.src = `data:image/png;base64, ${imgBase64Str}`;
			await new Promise((resolve) => (imgEl.onload = () => resolve(imgEl)));
			return imgEl;
		  })
		)) as CanvasImageSource[];
		outputImgs.push(initialSketchBitmap);
  
		outputFiles = {
		  sketch,
		  generations: (await Promise.all(
			imagesBase64Strs.map(async (imgBase64Str) => {
			  const dataUrl = `data:image/jpeg;base64, ${imgBase64Str}`;
			  const res = await fetch(dataUrl);
			  const blob = await res.blob();
			  const imgId = Date.now() % 200;
			  const fileName = `diffuse-the-rest-${imgId}.jpeg`;
			  return new File([blob], fileName, { type: 'image/jpeg' });
			})
		  )) as File[],
		};
  
		isShowSketch = true;
		let i = 0;
		imageTs = performance.now();
		drawImage(outputImgs[i % outputImgs.length]);
		drawNextImage = () => {
		  if (interval) clearInterval(interval);
		  imageTs = performance.now();
		  i += 1;
		  drawImage(outputImgs[i % outputImgs.length]);
		};
		interval = setInterval(() => {
		  i += 1;
		  imageTs = performance.now();
		  drawImage(outputImgs[i % outputImgs.length]);
		}, 2500);
  
		if (!isOutputControlAdded) addOutputControl();
		isSuccessfulGeneration = true;
	  } catch (err) {
		console.error(err);
		alert('');
	  } finally {
		isLoading = false;
	  }
	}
  
	function addOutputControl() {
	  const div = document.createElement('div');
	  div.className = 'drawing-board-control';
  
	  const btn = document.createElement('button');
	  btn.innerHTML = '⏯';
	  btn.onclick = drawNextImage;
	  div.append(btn);
  
	  const controlsEl = document.querySelector('.drawing-board-controls');
	  if (controlsEl && outputImgs.length > 1) {
		controlsEl.appendChild(div);
		isOutputControlAdded = true;
		canvasContainerEl.onclick = () => {
		  if (interval) clearInterval(interval);
		};
	  }
	}
  
	function addClearCanvasControl() {
	  const div = document.createElement('div');
	  div.className = 'drawing-board-control';
  
	  const btn = document.createElement('button');
	  btn.innerHTML = '🧹';
	  btn.onclick = () => {
		ctx?.clearRect(0, 0, canvasSize, canvasSize);
		outputImgs = [];
		isShowSketch = false;
	  };
	  div.append(btn);
  
	  const controlsEl = document.querySelector('.drawing-board-controls');
	  if (controlsEl) controlsEl.appendChild(div);
	}
  
	function addDownloadCanvasControl() {
	  const div = document.createElement('div');
	  div.className = 'drawing-board-control';
  
	  const btn = document.createElement('button');
	  btn.innerHTML = '⬇️';
	  btn.onclick = () => {
		if (!canvas) return;
		const link = document.createElement('a');
		const imgId = Date.now() % 200;
		link.download = `diffuse-the-rest-${imgId}.png`;
		link.href = canvas.toDataURL();
		link.click();
	  };
	  div.append(btn);
  
	  const controlsEl = document.querySelector('.drawing-board-controls');
	  if (controlsEl) controlsEl.appendChild(div);
	}
  
	function copySketch() {
	  const context = sketchEl.getContext('2d');
	  sketchEl.width = canvas.width;
	  sketchEl.height = canvas.height;
	  context!.drawImage(canvas, 0, 0);
	}
  
	async function drawUploadedImg(file: File) {
	  if (interval) clearInterval(interval);
	  const imgEl = new Image();
	  imgEl.src = URL.createObjectURL(file);
	  await new Promise((resolve) => (imgEl.onload = () => resolve(imgEl)));
	  const { width, height } = imgEl;
	  if (width === height) {
		ctx?.drawImage(imgEl, 0, 0, width, height, 0, 0, canvasSize, canvasSize);
	  } else if (width > height) {
		const canvasHeight = Math.floor((canvasSize * height) / width);
		const padding = Math.floor((canvasSize - canvasHeight) / 2);
		ctx?.drawImage(imgEl, 0, 0, width, height, 0, padding, canvasSize, canvasHeight);
	  } else {
		const canvasWidth = Math.floor((canvasSize * width) / height);
		const padding = Math.floor((canvasSize - canvasWidth) / 2);
		ctx?.drawImage(imgEl, 0, 0, width, height, padding, 0, canvasWidth, canvasSize);
	  }
	}
  
	function onfImgUpload() {
	  const file = fileInput.files?.[0];
	  if (file) drawUploadedImg(file);
	}
  
	function handleDrop(e: DragEvent) {
	  if (!e.dataTransfer?.files) return;
	  e.preventDefault();
	  const files = Array.from(e.dataTransfer.files);
	  const file = files[0];
	  drawUploadedImg(file);
	}
  
	function handlePaste(e: ClipboardEvent) {
	  if (!e.clipboardData) return;
	  const files = Array.from(e.clipboardData.files);
	  if (files.length === 0) return;
	  e.preventDefault();
	  const file = files[0];
	  drawUploadedImg(file);
	}
  
	function onKeyDown(e: KeyboardEvent) {
	  if (isLoading) return e.preventDefault();
	  if (e.code === 'Enter') {
		e.preventDefault();
		submitRequest();
	  }
	}
  
	function polyfillCreateImageBitmap() {
	  window.createImageBitmap = async function (data: ImageData): Promise<ImageBitmap> {
		return new Promise((resolve) => {
		  const canvas = document.createElement('canvas');
		  const ctx = canvas.getContext('2d');
		  canvas.width = data.width;
		  canvas.height = data.height;
		  ctx!.putImageData(data, 0, 0);
		  const dataURL = canvas.toDataURL();
		  const img = document.createElement('img');
		  img.addEventListener('load', () => resolve(img as any as ImageBitmap));
		  img.src = dataURL;
		});
	  };
	}
  
	function makeLinksTargetBlank() {
	  const linkEls = document.querySelectorAll('a');
	  for (const linkEl of linkEls) linkEl.target = '_blank';
	}
  
	async function uploadFile(file: File): Promise<string> {
	  const UPLOAD_URL = "https://huggingface.co/uploads";
	  const response = await fetch(UPLOAD_URL, {
		method: "POST",
		headers: {
		  "Content-Type": file.type,
		  "X-Requested-With": "XMLHttpRequest",
		},
		body: file,
	  });
	  return await response.text();
	}
  
	async function createCommunityPost() {
	  isUploading = true;
  
	  const files = [outputFiles.sketch, ...outputFiles.generations];
	  const urls = await Promise.all(files.map((f) => uploadFile(f)));
	  const htmlImgs = urls.map((url) => `<img src="${url}" width="400" height="400">`);
  
	  // Sketch and Generations display (no text labels)
	  const sketchDisplay = `<div style="display: flex; overflow: scroll; column-gap: 0.75rem;">${htmlImgs[0]}</div>`;
	  const generationsDisplay = `<div style="display: flex; flex-wrap: wrap; column-gap: 0.75rem;">${htmlImgs.slice(1).join("\n")}</div>`;
  
	  const descriptionMd = `${sketchDisplay}\n${generationsDisplay}`;
  
	  const params = new URLSearchParams({
		title: promptTxt,
		description: descriptionMd,
	  });
  
	  const paramsStr = params.toString();
	  window.open(`https://huggingface.co/spaces/huggingface-projects/diffuse-the-rest/discussions/new?${paramsStr}`, '_blank');
	  isUploading = false;
	}
  
	onMount(async () => {
	  if (typeof createImageBitmap === 'undefined') polyfillCreateImageBitmap();
	  const { innerWidth: windowWidth } = window;
	  canvasSize = Math.min(canvasSize, Math.floor(windowWidth * 0.75));
	  canvasContainerEl.style.width = `${canvasSize}px`;
	  canvasContainerEl.style.height = `${canvasSize}px`;
	  sketchEl.style.width = `${canvasSize}px`;
	  sketchEl.style.height = `${canvasSize}px`;
	  await tick();
	  drawingBoard = new window.DrawingBoard.Board('board-container', {
		size: 10,
		controls: ['Color', { Size: { type: 'dropdown' } }, { DrawingMode: { filler: false } }],
		webStorage: false,
		enlargeYourContainer: true,
	  });
	  canvas = drawingBoard.canvas;
	  ctx = canvas.getContext('2d');
	  canvas.ondragover = (e) => {
		e.preventDefault();
		return false;
	  };
	  addClearCanvasControl();
	  addDownloadCanvasControl();
	  makeLinksTargetBlank();
	});
  </script>
  
  <svelte:head>
	<link href="https://cdnjs.cloudflare.com/ajax/libs/drawingboard.js/0.4.2/drawingboard.css" rel="stylesheet" />
	<script src="https://code.jquery.com/jquery-1.12.4.min.js"></script>
	<script src="https://cdnjs.cloudflare.com/ajax/libs/drawingboard.js/0.4.2/drawingboard.min.js"></script>
	<script src="https://cdnjs.cloudflare.com/ajax/libs/iframe-resizer/4.3.1/iframeResizer.contentWindow.min.js"></script>
  </svelte:head>
  
  <svelte:window on:drop|preventDefault|stopPropagation={handleDrop} on:paste={handlePaste} />
  
  <div class="flex flex-wrap gap-x-4 gap-y-2 justify-center my-8">
	<canvas class="border-[1.2px] desktop:mt-[34px] {!isShowSketch ? 'hidden' : ''}" bind:this={sketchEl} />
	<div class="flex flex-col items-center {isLoading ? 'pointer-events-none' : ''}">
	  {#if !canvas}
		<div></div>
	  {/if}
	  <div id="board-container" bind:this={canvasContainerEl} />
	  {#if canvas}
		<div>
		  <div class="flex gap-x-2 mt-3 items-start justify-center align-vertical">
			<span
			  class="overflow-auto resize-y py-2 px-3 min-h-[42px] max-h-[500px] !w-[181px] whitespace-pre-wrap inline-block border border-gray-200 shadow-inner outline-none"
			  role="textbox"
			  contenteditable
			  spellcheck="false"
			  dir="auto"
			  maxlength="200"
			  bind:textContent={strength}
			  on:keydown={onKeyDown}
			/>
		  </div>
		  <div class="flex gap-x-2 mt-3 items-start justify-center">
			<span
			  class="overflow-auto resize-y py-2 px-3 min-h-[42px] max-h-[500px] !w-[181px] whitespace-pre-wrap inline-block border border-gray-200 shadow-inner outline-none"
			  role="textbox"
			  contenteditable
			  style="--placeholder: 'Add prompt'"
			  spellcheck="false"
			  dir="auto"
			  maxlength="1000"
			  bind:textContent={promptTxt}
			  on:keydown={onKeyDown}
			/>
		  </div>
		  <div class="flex gap-x-2 mt-3 items-start justify-center {isLoading ? 'animate-pulse' : ''}">
			<button
			  on:click={submitRequest}
			  class="bg-green-700 hover:bg-green-800 text-white font-bold py-[0.555rem] px-4 rounded-xl"
			>
			  diffuse 🪄
			</button>
		  </div>
		  <div class="mt-4">
			<label class="inline border py-2 px-3 bg-slate-200 cursor-pointer">
			  <input
				accept="image/*"
				bind:this={fileInput}
				on:change={onfImgUpload}
				style="display: none;"
				type="file"
			  />
			</label>
		  </div>
		  <div></div>
		</div>
	  {/if}
	</div>
  </div>
  
  <article class="prose-sm px-4 md:px-12 lg:px-56 mb-8 {!canvas ? 'hidden' : ''}">
	<div class="text-center"></div>
  </article>
  
  <style>
	span[contenteditable]:empty::before {
	  content: var(--placeholder);
	  color: rgba(156, 163, 175);
	}
  </style>