<script>
	import { onMount, createEventDispatcher } from "svelte";
	import * as Table from "$lib/components/ui/table";
	import * as Select from "$lib/components/ui/select";
	import { Separator } from "$lib/components/ui/separator";
	import Input from "./ui/input/input.svelte";

	import { read, utils } from "xlsx";
	import { ArrowDown, ArrowDownUp, ArrowUp, Copy, SearchX, Trash } from "lucide-svelte";
	import Button from "./ui/button/button.svelte";
	import { toast } from "svelte-sonner";

	export let file;
	const dispatch = createEventDispatcher();

	/* Init Variables */
	let workbooks;
	let selectedWBIndex;
	let data = [];
	let filteredData = [];
	let header;

	/* Search Variables */
	let searchString = "";

	/* Sort Variables */
	let sortedSequence = [-1, 0, 1];
	let sortedType = 0;
	let sortedBy;

	/* Cell Variables */
	let selectedCell;
	let cellValue = "";

	onMount(async () => {
		const f = await file.arrayBuffer();
		const wb = read(f);

		workbooks = wb.SheetNames;
		if (wb.SheetNames.length == 1) {
			handleWorkbookSelection(0);
		}
	});

	function handleFileDelete() {
		dispatch("deleteFile");
	}

	/* TODO: #6 Create a Progress bar or loading indicator */
	async function loadXLSX(index) {
		const f = await file.arrayBuffer();
		const wb = read(f);
		workbooks = wb.SheetNames;
		const ws = wb.Sheets[wb.SheetNames[index]];

		// Bestimme den Bereich des Arbeitsblatts
		let range = utils.decode_range(ws["!ref"]);

		// Überprüfe die ersten Zeilen auf eine gültige Kopfzeile
		const maxRowsToCheck = 10; // Überprüfe die ersten 10 Zeilen auf eine nicht-leere Kopfzeile
		let validHeaderRowIndex = 0;

		for (let i = 0; i < maxRowsToCheck; i++) {
			const rowData = utils.sheet_to_json(ws, { header: 1, range: i, raw: false })[0];

			// Überprüfe, ob diese Zeile mehr als eine nicht-leere Zelle enthält
			if (rowData && rowData.filter((cell) => cell && cell.trim() !== "").length > 1) {
				if (i > 0) {
					toast.info(`We found a valid header row in row ${i + 1}!`);
				}
				validHeaderRowIndex = i;
				break;
			}
		}

		// Setze den Start des Bereichs auf die gefundene Kopfzeile
		range.s.r = validHeaderRowIndex;
		ws["!ref"] = utils.encode_range(range);

		// Lese die Daten basierend auf dem angepassten Bereich
		data = utils.sheet_to_json(ws);

		// Extrahiere die Kopfzeile und setze sie
		header = ["Row", ...Object.keys(data[0])];

		// Fügt eine aufsteigende Nummer zu den Daten hinzu
		data = data.map((row, index) => {
			return { Row: index + 1, ...row };
		});

		filteredData = data; // Initialisiere die gefilterten Daten mit allen Daten
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

	function handleSort(head) {
		sortedBy = head;
		const currentSortIndex = sortedSequence.indexOf(sortedType);
		if (currentSortIndex === -1) {
			throw new Error("Invalid sort type");
		}

		const nextIndex = (currentSortIndex + 1) % sortedSequence.length;
		sortedType = sortedSequence[nextIndex];
		if (sortedType === 0) {
			// Reset to original order
			filteredData = [...data];
		} else {
			filteredData = [...data].sort((a, b) => {
				const aValue = a[sortedBy];
				const bValue = b[sortedBy];

				if (aValue > bValue) return sortedType;
				if (aValue < bValue) return -sortedType;
				return 0;
			});
		}
	}

	function highlightMatch(text, query) {
		const regex = new RegExp(`(${query})`, "gi");
		return text.replace(regex, '<span class="p-[2px] rounded-sm shadow-sm bg-primary/40">$1</span>');
	}

	function copyRow(row) {
		let copyButton = document.getElementById("copy" + row.Row);
		toast.info(`Row ${row.Row} copied`);
		copyButton.classList.add("animate-pulse");

		/* Copy to clipboard and skip row column */
		const rowText = Object.values(row)
			.slice(1)
			.map((value) => `"${value}"`)
			.join("\t");
		navigator.clipboard.writeText(rowText);

		/* 2 second delay */
		setTimeout(() => {
			copyButton.classList.remove("animate-pulse");
		}, 2000);
	}

	function handleCellSelection(e) {
		if (selectedCell != e.target.id && selectedCell) {
			document.getElementById(selectedCell).classList.remove("border-2", "border-primary/30", "bg-primary/10");
		}
		selectedCell = e.target.id;

		document.getElementById(e.target.id).classList.add("border-2", "border-primary/30", "bg-primary/10");

		cellValue = e.target.innerHTML;
	}

	/* Eventlistener on CTRL+C */
	window.addEventListener("keydown", (e) => {
		if (e.ctrlKey && e.key === "c" && cellValue) {
			e.preventDefault();
			navigator.clipboard.writeText(cellValue);
		}
	});
</script>

<div>
	<div class="flex flex-row items-center justify-between px-4">
		<h1 class="py-4 text-xl tracking-tight scroll-m-20">{file.name}</h1>
		<Button on:click={handleFileDelete} variant="ghost" class="hover:bg-red-500 hover:text-white"><Trash class="w-4 h-4" /></Button>
	</div>
	{#if workbooks && workbooks.length > 1}
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
	{/if}

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
					{#each header as head}
						<Table.Head>
							<div class="flex flex-row items-center gap-2">
								{head}
								{#if sortedBy != ""}
									<Button variant="ghost" on:click={() => handleSort(head)}>
										{#if sortedType === 0}
											<ArrowDownUp class="w-4 h-4" />
										{:else if sortedType === -1 && sortedBy === head}
											<ArrowDown class="w-4 h-4" />
										{:else if sortedType === 1 && sortedBy === head}
											<ArrowUp class="w-4 h-4" />
										{/if}
									</Button>
								{/if}
							</div>
						</Table.Head>
					{/each}
				</Table.Row>
			</Table.Header>

			<Table.Body>
				{#each filteredData as row}
					<Table.Row class="items-center" id="copy{row.Row}">
						<Table.Cell><Button class="duration-[1ms]" variant="ghost" on:click={() => copyRow(row)}><Copy class="w-4 h-4" /></Button></Table.Cell>
						{#each header as head}
							<Table.Cell id="cell_{head}_{row.Row}" on:click={(e) => handleCellSelection(e)}
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
