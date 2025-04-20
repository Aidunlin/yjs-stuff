<script lang="ts">
	import { generateKeyBetween, generateNKeysBetween } from 'fractional-indexing';
	import { onMount } from 'svelte';
	import { flip } from 'svelte/animate';
	import { IndexeddbPersistence } from 'y-indexeddb';
	import { WebrtcProvider } from 'y-webrtc';
	import * as Y from 'yjs';

	const SIGNAL_SERVER = 'https://y-webrtc-4ou5.onrender.com';
	const DOC_ID = 'my-document-id';
	const RANKS_NAME = 'ranks';
	const CHECKED_NAME = 'checked';
	const INDEXES_NAME = 'indexes';

	const flipDurationMs = 100;

	type ListItem = {
		team: string;
		index: string;
		rank: number | undefined;
		checked: boolean;
	};

	let doc: Y.Doc;
	let sharedTeamDetails: Y.Map<{ rank: number }>;
	let sharedTeamIndexes: Y.Map<string>;
	let sharedTeamChecks: Y.Map<true>;

	let undoManager = $state<Y.UndoManager>();
	let canUndo = $state(false);
	let canRedo = $state(false);

	let idb: IndexeddbPersistence;
	let webrtc: WebrtcProvider;

	let viewTeamDetails = $state(new Map<string, { rank: number }>());
	let viewTeamIndexes = $state(new Map<string, string>());
	let viewTeamChecks = $state(new Set<string>());

	function createListView(
		indexes: Map<string, string>,
		details: Map<string, { rank: number }>,
		checks: Set<string>
	) {
		return Array.from(indexes)
			.map(
				([team, index]): ListItem => ({
					team,
					index,
					rank: details.get(team)?.rank,
					checked: checks.has(team)
				})
			)
			.toSorted(sortListItems);
	}

	let listView = $derived(createListView(viewTeamIndexes, viewTeamDetails, viewTeamChecks));

	let activeTeams = $derived(listView.filter((r) => !r.checked));
	let crossedOffTeams = $derived(listView.filter((r) => r.checked));

	let alive = $state(!!localStorage.getItem('alive'));

	let moving = $state<{ item: ListItem; arrIndex: number } | undefined>();

	onMount(() => {
		alive && start();
	});

	function start() {
		doc = new Y.Doc();
		sharedTeamDetails = doc.getMap(RANKS_NAME);
		sharedTeamChecks = doc.getMap(CHECKED_NAME);
		sharedTeamIndexes = doc.getMap(INDEXES_NAME);

		idb = new IndexeddbPersistence(DOC_ID, doc);
		webrtc = new WebrtcProvider(DOC_ID, doc, { signaling: [SIGNAL_SERVER] });

		undoManager = new Y.UndoManager(doc);
		undoManager.on('stack-item-added', updateCanUndo);
		undoManager.on('stack-item-popped', updateCanUndo);
		undoManager.on('stack-cleared', updateCanUndo);

		alive = true;
		localStorage.setItem('alive', 'true');

		sharedTeamDetails.observe(() => {
			viewTeamDetails = new Map(sharedTeamDetails.entries());

			if (moving) {
				moving.item.rank = viewTeamDetails.get(moving.item.team)?.rank ?? moving.item.rank;
			}
		});

		sharedTeamChecks.observe(() => {
			viewTeamChecks = new Set(sharedTeamChecks.keys());

			if (moving) {
				if (viewTeamChecks.has(moving.item.team)) {
					moving = undefined;
				} else {
					activeTeams = createListView(viewTeamIndexes, viewTeamDetails, viewTeamChecks).filter(
						(r) => !r.checked
					);

					moving.arrIndex = activeTeams.findIndex((r) => r.team == moving!.item.team);
				}
			}
		});

		sharedTeamIndexes.observe(() => {
			viewTeamIndexes = new Map(sharedTeamIndexes.entries());

			if (moving) {
				activeTeams = createListView(viewTeamIndexes, viewTeamDetails, viewTeamChecks).filter(
					(r) => !r.checked
				);

				moving.arrIndex = activeTeams.findIndex((r) => r.team == moving!.item.team);
				moving.item.index = viewTeamIndexes.get(moving.item.team) ?? moving.item.index;
			}
		});
	}

	function updateCanUndo() {
		canUndo = undoManager?.canUndo() ?? false;
		canRedo = undoManager?.canRedo() ?? false;
	}

	function stop() {
		alive = false;
		localStorage.removeItem('alive');

		doc.destroy();
		idb.destroy();
		webrtc.destroy();
		undoManager?.destroy();

		canUndo = false;
		canRedo = false;

		viewTeamDetails = new Map();
		viewTeamIndexes = new Map();
		viewTeamChecks = new Set();
	}

	function clear() {
		stop();
		idb.clearData();
	}

	function fill() {
		const defaultRanks = [
			{ rank: 1, team: '2910' },
			{ rank: 2, team: '1778' },
			{ rank: 3, team: '2930' },
			{ rank: 4, team: '9450' },
			{ rank: 5, team: '3663' },
			{ rank: 6, team: '2046' },
			{ rank: 7, team: '9442' },
			{ rank: 8, team: '1540' }
		];

		const indexes = generateNKeysBetween(null, null, defaultRanks.length);

		doc.transact(() => {
			defaultRanks.forEach(({ rank, team }, index) => {
				if (!viewTeamDetails.has(team)) {
					sharedTeamDetails.set(team, { rank });
				}
				if (!viewTeamIndexes.has(team)) {
					sharedTeamIndexes.set(team, indexes[index]);
				}
			});
		});
	}

	function move(toIndex: number) {
		if (!moving) throw new Error('no moving data');

		const fromIndex = moving.arrIndex;
		const diff = toIndex - fromIndex;

		if (diff == 0) {
			throw new Error('diff is 0');
		}

		let startArrIndex: number | undefined;
		let endArrIndex: number | undefined;

		if (diff > 0) {
			startArrIndex = toIndex;
			endArrIndex = toIndex + 1;

			if (startArrIndex >= activeTeams.length) startArrIndex = undefined;
			if (endArrIndex >= activeTeams.length) endArrIndex = undefined;
		} else {
			startArrIndex = toIndex - 1;
			endArrIndex = toIndex;

			if (startArrIndex < 0) startArrIndex = undefined;
			if (endArrIndex < 0) endArrIndex = undefined;
		}

		let startIndex =
			startArrIndex !== undefined ? $state.snapshot(activeTeams[startArrIndex]).index : undefined;
		let endIndex =
			endArrIndex !== undefined ? $state.snapshot(activeTeams[endArrIndex]).index : undefined;

		sharedTeamIndexes.set(moving.item.team, generateKeyBetween(startIndex, endIndex));
		moving = undefined;
	}

	function getOrdinal(n: number) {
		if (n % 10 == 1 && n % 100 != 11) return 'st';
		if (n % 10 == 2 && n % 100 != 12) return 'nd';
		if (n % 10 == 3 && n % 100 != 13) return 'rd';
		return 'th';
	}

	function sortListItems(a: ListItem, b: ListItem) {
		if (a.index < b.index) return -1;
		if (a.index > b.index) return 1;
		return 0;
	}
</script>

<h1>yjs-stuff</h1>

{#if alive}
	<p>
		<button onclick={stop}>stop</button>
		<button onclick={clear}>clear and stop</button>
		<button onclick={fill}>fill</button>
	</p>

	<p>
		<button onclick={() => undoManager?.undo()} disabled={!canUndo}>undo</button>
		<button onclick={() => undoManager?.redo()} disabled={!canRedo}>redo</button>
	</p>

	<div class="grid">
		{#if activeTeams.length}
			<div style="grid-column: span 4">Teams</div>

			{#each activeTeams as row, index (row.team)}
				{@const thisMoving = moving?.item.team == row.team}
				{@const bgColor = thisMoving ? 'darkgray' : 'lightgray'}

				<div
					animate:flip={{ duration: flipDurationMs }}
					class="item"
					style="background-color:{bgColor}"
				>
					<input
						type="checkbox"
						disabled={!!moving}
						onchange={() => sharedTeamChecks.set(row.team, true)}
					/>
					<div>{row.rank}{getOrdinal(Number(row.rank))}</div>
					<div>{row.team}</div>
					<button
						onclick={() => {
							if (!moving) {
								moving = { item: row, arrIndex: index };
							} else if (thisMoving) {
								moving = undefined;
							} else {
								move(index);
							}
						}}
						style="width:30px;height:24px"
					>
						{#if !moving}
							{#if index == 0}
								&DownArrow;
							{:else if index == activeTeams.length - 1}
								&UpArrow;
							{:else}
								&UpDownArrow;
							{/if}
						{:else if thisMoving}
							x
						{:else if moving.arrIndex > index}
							&lsh;
						{:else if moving.arrIndex < index}
							&ldsh;
						{/if}
					</button>
				</div>
			{/each}
		{/if}

		{#if crossedOffTeams.length}
			<div style="grid-column: span 4">Crossed off</div>

			{#each crossedOffTeams as { rank, team } (team)}
				<div class="item">
					<input type="checkbox" checked onchange={() => sharedTeamChecks.delete(team)} />
					<div>{rank}{getOrdinal(Number(rank))}</div>
					<div>{team}</div>
				</div>
			{/each}
		{/if}
	</div>
{:else}
	<p>
		<button onclick={start}>start</button>
	</p>
{/if}

<style>
	.grid {
		display: grid;
		grid-template-columns: repeat(4, min-content);
		align-items: center;
		gap: 4px;
	}

	.item {
		display: grid;
		grid-template-columns: subgrid;
		align-items: center;
		grid-column: span 4;
		padding: 2px;
	}
</style>
