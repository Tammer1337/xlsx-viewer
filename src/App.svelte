<script>
	import { Separator } from "$lib/components/ui/separator";
	import { Moon, Sun, Table2, Upload } from "lucide-svelte";
	import { ModeWatcher, toggleMode } from "mode-watcher";
	import { Button } from "$lib/components/ui/button/index.js";
	import { Toaster } from "$lib/components/ui/sonner";
	import { toast } from "svelte-sonner";
	import XlsxViewer from "$lib/components/XLSXViewer.svelte";

	let file;

	function handleFiles(files) {
		if (files[0].type == "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet") {
			toast.success("File successfully loaded");

			file = files[0];
		} else {
			toast.error("Invalid file type", { description: "Only .xlsx files are supported" });
		}
	}

	function triggerFileInput() {
		const fileElem = document.getElementById("fileElem");
		if (fileElem) {
			fileElem.click();
		}
	}

	function handleFileSelect(event) {
		const input = event.target;
		if (input && input.files) {
			handleFiles(input.files);
		}
	}

	function handleDrop(event) {
		event.preventDefault();
		event.stopPropagation();
		const dropArea = document.getElementById("drop-area");
		if (dropArea) {
			dropArea.classList.remove("bg-primary-foreground/50");
		}

		const files = event.dataTransfer?.files;
		if (files) {
			handleFiles(files);
		}
	}

	function handleDragOver(event) {
		event.preventDefault();
		event.stopPropagation();
		const dropArea = document.getElementById("drop-area");
		if (dropArea) {
			dropArea.classList.add("bg-primary-foreground/50");
		}
	}

	function handleDragLeave(event) {
		event.preventDefault();
		event.stopPropagation();
		const dropArea = document.getElementById("drop-area");
		if (dropArea) {
			dropArea.classList.remove("bg-primary-foreground/50");
		}
	}
</script>

<Toaster richColors closeButton />
<div class="h-screen p-4 space-y-5 bg-background">
	<div class="flex items-center justify-between">
		<a href="/" class="flex items-center p-2 space-x-2 rounded-md hover:bg-foreground/10 hover:cursor-pointer">
			<Table2 class="w-5 h-5 shadow-xl text-primary" />
			<h1 class="text-2xl font-bold">XLSX Viewer</h1>
		</a>
		<ModeWatcher />
		<Button on:click={toggleMode} variant="outline" size="icon">
			<Sun class="h-[1.2rem] w-[1.2rem] rotate-0 scale-100 transition-all dark:-rotate-90 dark:scale-0" />
			<Moon class="absolute h-[1.2rem] w-[1.2rem] rotate-90 scale-0 transition-all dark:rotate-0 dark:scale-100" />
			<span class="sr-only">Toggle theme</span>
		</Button>
	</div>
	<Separator class="my-4" />
	<!-- TODO: #3 Destroy current file from load -->
	{#if !file}
		<div
			id="drop-area"
			role="button"
			tabindex="0"
			class="flex items-center justify-center flex-1 p-4 border border-dashed rounded-lg shadow-sm min-h-56 border-foreground/30"
			on:drop={handleDrop}
			on:dragover={handleDragOver}
			on:dragleave={handleDragLeave}
		>
			<form class="my-form"></form>
			<div class="flex flex-col items-center gap-1 text-center">
				<h3 class="text-xl tracking-tight">Drag and drop an XLSX File here</h3>
				<input class="hidden" type="file" id="fileElem" accept=".xlsx" on:change={handleFileSelect} />
				<Button class="mt-4" on:click={triggerFileInput}>
					<Upload class="w-4 h-4 mr-2" />
					Load XLSX
				</Button>
			</div>
		</div>
	{:else}
		<XlsxViewer {file} />
	{/if}
</div>
