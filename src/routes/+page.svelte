<script lang="ts">
	import { generateNJitteredKeysBetween, IndexGenerator } from 'fractional-indexing-jittered';
	import { onMount } from 'svelte';
	import { flip } from 'svelte/animate';
	import { IndexeddbPersistence } from 'y-indexeddb';
	import { WebrtcProvider } from 'y-webrtc';
	import * as Y from 'yjs';

	const SIGNAL_SERVER = 'https://y-webrtc-4ou5.onrender.com';
	const DETAILS_NAME = 'details';
	const CHECKED_NAME = 'checked';
	const INDEXES_NAME = 'indexes';

	const flipDurationMs = 100;

	const indexGen = new IndexGenerator([]);

	type ListItem = {
		team: string;
		index: string;
		rank: number;
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

	let listView = $derived(createListView());

	let activeTeams = $derived(listView.filter((r) => !r.checked));
	let crossedOffTeams = $derived(listView.filter((r) => r.checked));

	let alive = $state(!!localStorage.getItem('alive'));

	let userName = $state(localStorage.getItem('username') ?? '');
	let docName = $state(localStorage.getItem('docname') ?? '');
	let signalingServerPassword = $state(localStorage.getItem('password') ?? '');

	let movingTeam = $state<string | undefined>();

	onMount(() => {
		alive && start();
	});

	function createListView() {
		return Array.from(viewTeamIndexes)
			.map(
				([team, index]): ListItem => ({
					team,
					index,
					rank: viewTeamDetails.get(team)?.rank ?? 0,
					checked: viewTeamChecks.has(team)
				})
			)
			.toSorted((a, b) => {
				if (a.index < b.index) return -1;
				if (a.index > b.index) return 1;
				if (a.rank < b.rank) return -1;
				if (a.rank > b.rank) return 1;
				return 0;
			});
	}

	function start() {
		if (!docName) {
			return;
		}

		localStorage.setItem('docname', docName);

		doc = new Y.Doc();
		sharedTeamDetails = doc.getMap(DETAILS_NAME);
		sharedTeamChecks = doc.getMap(CHECKED_NAME);
		sharedTeamIndexes = doc.getMap(INDEXES_NAME);

		idb = new IndexeddbPersistence(docName, doc);
		webrtc = new WebrtcProvider(docName, doc, {
			signaling: [SIGNAL_SERVER],
			password: signalingServerPassword || undefined
		});

		undoManager = new Y.UndoManager(doc);
		undoManager.on('stack-item-added', updateCanUndo);
		undoManager.on('stack-item-popped', updateCanUndo);

		alive = true;
		localStorage.setItem('alive', 'true');
		signalingServerPassword && localStorage.setItem('password', signalingServerPassword);

		if (userName) {
			localStorage.setItem('username', userName);
			webrtc.awareness.setLocalStateField('user', userName);
		}

		sharedTeamDetails.observe(() => {
			viewTeamDetails = new Map(sharedTeamDetails.entries());
		});

		sharedTeamChecks.observe(() => {
			viewTeamChecks = new Set(sharedTeamChecks.keys());

			if (movingTeam && viewTeamChecks.has(movingTeam)) {
				movingTeam = undefined;
			}
		});

		sharedTeamIndexes.observe(() => {
			viewTeamIndexes = new Map(sharedTeamIndexes.entries());
			indexGen.updateList(Array.from(viewTeamIndexes.values()));
		});
	}

	function updateCanUndo() {
		canUndo = undoManager?.canUndo() ?? false;
		canRedo = undoManager?.canRedo() ?? false;
	}

	function stop() {
		alive = false;
		localStorage.removeItem('alive');

		indexGen.updateList([]);

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

		const indexes = generateNJitteredKeysBetween(null, null, defaultRanks.length);

		doc.transact(() => {
			sharedTeamDetails.clear();
			sharedTeamIndexes.clear();
			sharedTeamChecks.clear();

			defaultRanks.forEach(({ team, rank }, i) => {
				sharedTeamDetails.set(team, { rank });
				sharedTeamIndexes.set(team, indexes[i]);
			});
		});
	}

	function move(toTeam: string) {
		removeAnyDuplicates();

		if (!movingTeam) {
			console.error('no moving data');
			return;
		}

		const toArrIndex = activeTeams.findIndex((r) => r.team == toTeam);
		const toIndex = activeTeams.find((r) => r.team == toTeam)?.index;

		if (!toIndex) {
			console.error(`could not find index to move ${movingTeam} to ${toTeam}`);
			movingTeam = undefined;
			return;
		}

		const fromArrIndex = activeTeams.findIndex((r) => r.team == movingTeam);
		const arrIndexDiff = toArrIndex - fromArrIndex;

		if (!arrIndexDiff) {
			console.error(`no diff between ${movingTeam} and ${toTeam}`);
			movingTeam = undefined;
			return;
		}

		if (arrIndexDiff > 0) {
			sharedTeamIndexes.set(movingTeam, indexGen.keyAfter(toIndex));
		} else {
			sharedTeamIndexes.set(movingTeam, indexGen.keyBefore(toIndex));
		}

		movingTeam = undefined;
	}

	function removeAnyDuplicates() {
		let duplicates = false;

		const set = new Set<string>();
		for (const index of listView.map((r) => r.index)) {
			if (set.has(index)) {
				duplicates = true;
				break;
			}

			set.add(index);
		}

		if (!duplicates) {
			return;
		}

		console.warn('duplicate indexes found, regenerating all indexes while keeping same order');

		const indexes = generateNJitteredKeysBetween(null, null, listView.length);

		doc.transact(() => {
			listView.forEach(({ team }, i) => {
				sharedTeamIndexes.set(team, indexes[i]);
			});
		});
	}

	function getOrdinal(n: number) {
		if (n % 10 == 1 && n % 100 != 11) return 'st';
		if (n % 10 == 2 && n % 100 != 12) return 'nd';
		if (n % 10 == 3 && n % 100 != 13) return 'rd';
		return 'th';
	}
</script>

{#if alive}
	<h1>
		{userName} | {docName} | {signalingServerPassword}
	</h1>

	<p>
		<button onclick={stop}>stop</button>
		<button onclick={clear}>clear and stop</button>
		<button onclick={fill}>reset data</button>
	</p>

	<p>
		<button onclick={() => undoManager?.undo()} disabled={!canUndo}>undo</button>
		<button onclick={() => undoManager?.redo()} disabled={!canRedo}>redo</button>
	</p>

	<div class="grid">
		{#if activeTeams.length}
			{@const movingIndex = activeTeams.find((r) => r.team == movingTeam)?.index}

			<div style="grid-column: span 5">Teams</div>

			{#each activeTeams as row, arrIndex (row.team)}
				{@const thisMoving = movingTeam == row.team}
				{@const bgColor = thisMoving ? 'darkgray' : 'lightgray'}

				<div
					animate:flip={{ duration: flipDurationMs }}
					class="item"
					style="background-color:{bgColor}"
				>
					<input
						type="checkbox"
						disabled={!!movingTeam}
						onchange={() => sharedTeamChecks.set(row.team, true)}
					/>
					<div>{row.rank}{getOrdinal(row.rank)}</div>
					<div>{row.team}</div>
					<button
						onclick={() => {
							if (!movingTeam) {
								movingTeam = row.team;
							} else if (thisMoving) {
								movingTeam = undefined;
							} else {
								move(row.team);
							}
						}}
						style="width:30px;height:24px"
					>
						{#if !movingTeam}
							{#if arrIndex == 0}
								&DownArrow;
							{:else if arrIndex == activeTeams.length - 1}
								&UpArrow;
							{:else}
								&UpDownArrow;
							{/if}
						{:else if thisMoving}
							x
						{:else if movingIndex && movingIndex > row.index}
							&lsh;
						{:else if movingIndex && movingIndex < row.index}
							&ldsh;
						{/if}
					</button>
					<div style="font-family:monospace">{row.index}</div>
				</div>
			{/each}
		{/if}

		{#if crossedOffTeams.length}
			<div style="grid-column: span 5">Crossed off</div>

			{#each crossedOffTeams as { rank, team, index } (team)}
				<div class="item">
					<input type="checkbox" checked onchange={() => sharedTeamChecks.delete(team)} />
					<div>{rank}{getOrdinal(rank)}</div>
					<div>{team}</div>
					<div></div>
					<div style="font-family:monospace">{index}</div>
				</div>
			{/each}
		{/if}
	</div>
{:else}
	<h1>yjs-stuff</h1>

	<p>
		<label>
			user name<br />
			<input bind:value={userName} />
		</label>
	</p>
	<p>
		<label>
			doc name<br />
			<input bind:value={docName} />
		</label>
	</p>
	<p>
		<label>
			lobby code<br />
			<input bind:value={signalingServerPassword} />
		</label>
	</p>
	<p>
		<button onclick={start}>start</button>
	</p>
{/if}

<style>
	.grid {
		display: grid;
		grid-template-columns: repeat(5, min-content);
		align-items: center;
		gap: 4px;
	}

	.item {
		display: grid;
		grid-template-columns: subgrid;
		align-items: center;
		grid-column: span 5;
		padding: 2px;
	}
</style>
