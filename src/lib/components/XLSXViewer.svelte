<script>
	import { onMount } from "svelte";
	import * as Table from "$lib/components/ui/table";
	import * as Select from "$lib/components/ui/select";
	import { Separator } from "$lib/components/ui/separator";
	import Input from "./ui/input/input.svelte";
	import { Checkbox } from "$lib/components/ui/checkbox/index.js";

	import { read, utils } from "xlsx";
	import { Copy, SearchX, Trash } from "lucide-svelte";
	import Button from "./ui/button/button.svelte";
	import { toast } from "svelte-sonner";

	export let file;
	let workbooks;
	let selectedWBIndex;
	let data = [];
	let filteredData = [];
	let header;
	let searchString = "";

	onMount(async () => {
		const f = await file.arrayBuffer();
		const wb = read(f);
		workbooks = wb.SheetNames;
	});

	async function loadXLSX(index) {
		const f = await file.arrayBuffer();
		const wb = read(f);
		workbooks = wb.SheetNames;
		const ws = wb.Sheets[wb.SheetNames[index]];
		data = utils.sheet_to_json(ws);
		header = Object.keys(data[0]);
		filteredData = data; // Initially, display all data
	}

	function handleWorkbookSelection(value) {
		selectedWBIndex = value;
		loadXLSX(selectedWBIndex);
	}

	function handleSearch(e) {
		searchString = e.target.value;
		if (searchString.length > 0) {
			filteredData = data.filter((row) => {
				return Object.keys(row).some((key) => {
					return String(row[key]).toLowerCase().includes(searchString.toLowerCase());
				});
			});
		} else {
			filteredData = data; // Reset to full data if search string is empty
		}
	}

	/* TODO: Sort functionality */

	function highlightMatch(text, query) {
		const regex = new RegExp(`(${query})`, "gi");
		return text.replace(regex, '<span class="p-[2px] rounded-sm shadow-sm bg-primary/40">$1</span>');
	}

	function copyRow(row, index) {
		let copyButton = document.getElementById("copy" + index);
		toast.info(`Row ${index + 1} copied`);
		copyButton.classList.add("animate-pulse");

		/* Copy to clipboard */
		const rowText = Object.values(row)
			.map((value) => `"${value}"`)
			.join("\t");
		navigator.clipboard.writeText(rowText);

		/* 2 second delay */
		setTimeout(() => {
			copyButton.classList.remove("animate-pulse");
		}, 2000);
	}

	/* TODO: Get Cell Selection */
	function handleCellSelection(e) {
		let target = e.target;

		const tdElements = document.querySelector(".border-2.border-primary/40.bg-foreground/10.shadow-lg");

		tdElements?.classList.remove("border-2", "border-primary/40", "bg-foreground/10", "shadow-lg");

		target.classList.add("border-2", "border-primary/40", "bg-foreground/10", "shadow-lg");
	}
</script>

<div>
	<div class="flex flex-row items-center justify-between">
		<h1 class="py-4 text-xl tracking-tight scroll-m-20">{file.name}</h1>
		<Button on:click variant="ghost" class="hover:bg-red-500 hover:text-white"><Trash class="w-4 h-4" /></Button>
	</div>
	<Select.Root onSelectedChange={(v) => handleWorkbookSelection(v.value)}>
		<Select.Trigger class="w-full ">
			<Select.Value placeholder="Select worksheet" />
		</Select.Trigger>
		<Select.Content>
			{#each workbooks as workbook, index}
				<Select.Item value={index}>{workbook}</Select.Item>
			{/each}
		</Select.Content>
	</Select.Root>

	<Separator class="mt-6" />

	{#if filteredData && selectedWBIndex >= 0}
		<div class="my-6">
			<Input on:keyup={(e) => handleSearch(e)} placeholder="Search" />
		</div>
	{/if}
	{#if filteredData.length > 0}
		<Table.Root>
			<Table.Header>
				<Table.Row>
					<Table.Head class="w-8"></Table.Head>
					<Table.Head class="w-8">Row</Table.Head>
					{#each header as head}
						<Table.Head>{head}</Table.Head>
					{/each}
				</Table.Row>
			</Table.Header>

			<Table.Body>
				{#each filteredData as row, i}
					<Table.Row class="items-center">
						<Table.Cell
							><Button class="duration-[1ms]" id="copy{i}" variant="ghost" on:click={() => copyRow(row, i)}><Copy class="w-4 h-4" /></Button></Table.Cell
						>
						<Table.Cell>{i + 1}</Table.Cell>
						{#each header as head}
							<Table.Cell id="cell{i}{row[head]}" on:click={(e) => handleCellSelection(e)}
								>{#if searchString.length > 0}
									{@html highlightMatch(String(row[head]), searchString)}
								{:else}
									{row[head]}
								{/if}</Table.Cell
							>
						{/each}
					</Table.Row>
				{/each}
			</Table.Body>
		</Table.Root>
	{:else}
		<div class="flex flex-col items-center justify-center h-56 text-gray-400 dark:text-gray-600">
			<SearchX class="w-10 h-10 " />
			<p>No data found!</p>
			<p>Please select a worksheet to display your table</p>
		</div>
	{/if}
</div>
