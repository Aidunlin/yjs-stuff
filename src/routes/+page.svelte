<script lang="ts">
	import {
		generateJitteredKeyBetween,
		generateNJitteredKeysBetween
	} from 'fractional-indexing-jittered';
	import { onMount } from 'svelte';
	import { flip } from 'svelte/animate';
	import { IndexeddbPersistence } from 'y-indexeddb';
	import { WebrtcProvider } from 'y-webrtc';
	import * as Y from 'yjs';

	const SIGNAL_SERVER = 'https://y-webrtc-4ou5.onrender.com';
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

	let listView = $derived(createListView());

	$inspect(listView.map((r) => r.index));

	let activeTeams = $derived(listView.filter((r) => !r.checked));
	let crossedOffTeams = $derived(listView.filter((r) => r.checked));

	let alive = $state(!!localStorage.getItem('alive'));

	let userName = $state(localStorage.getItem('username') ?? '');
	let docName = $state(localStorage.getItem('docname') ?? '');
	let signalingServerPassword = $state(localStorage.getItem('password') ?? '');

	let moving = $state<{ team: string; index: string } | undefined>();

	onMount(() => {
		alive && start();
	});

	function createListView() {
		return Array.from(viewTeamIndexes)
			.map(
				([team, index]): ListItem => ({
					team,
					index,
					rank: viewTeamDetails.get(team)?.rank,
					checked: viewTeamChecks.has(team)
				})
			)
			.toSorted((a, b) => {
				if (a.index < b.index) return -1;
				if (a.index > b.index) return 1;
				if (a.rank && b.rank && a.rank < b.rank) return -1;
				if (a.rank && b.rank && a.rank > b.rank) return 1;
				return 0;
			});
	}

	function start() {
		if (!docName) {
			return;
		}

		localStorage.setItem('docname', docName);

		doc = new Y.Doc();
		sharedTeamDetails = doc.getMap(RANKS_NAME);
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
		undoManager.on('stack-cleared', updateCanUndo);

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

			if (moving && viewTeamChecks.has(moving.team)) {
				moving = undefined;
			}
		});

		sharedTeamIndexes.observe(() => {
			viewTeamIndexes = new Map(sharedTeamIndexes.entries());

			if (moving) {
				moving.index = viewTeamIndexes.get(moving.team) ?? moving.index;
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

		const indexes = generateNJitteredKeysBetween(null, null, defaultRanks.length);

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

		const fromIndex = activeTeams.findIndex((r) => r.team == moving!.team);
		const diff = toIndex - fromIndex;

		if (fromIndex === -1 || diff == 0) {
			alert('Something went terribly wrong!');
			moving = undefined;
			return;
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

		const startIndex = startArrIndex !== undefined ? activeTeams[startArrIndex].index : null;
		const endIndex = endArrIndex !== undefined ? activeTeams[endArrIndex].index : null;

		if (startIndex === endIndex && startArrIndex !== undefined && endArrIndex !== undefined) {
			// To move an item between too conflicting indexes, we need to replace them with new ones.

			const startTeam = activeTeams[startArrIndex];
			const endTeam = activeTeams[endArrIndex];

			const beforeArrIndex = startArrIndex == 0 ? undefined : startArrIndex - 1;
			const afteArrIndex = endArrIndex == activeTeams.length - 1 ? undefined : endArrIndex + 1;

			const beforeIndex = beforeArrIndex !== undefined ? activeTeams[beforeArrIndex].index : null;
			const afterIndex = afteArrIndex !== undefined ? activeTeams[afteArrIndex].index : null;

			if (beforeIndex === afterIndex) {
				alert('Something went terribly wrong!');
				moving = undefined;
				return;
			}

			const [newStartIndex, newIndex, newEndIndex] = generateNJitteredKeysBetween(
				beforeIndex,
				afterIndex,
				3
			);

			doc.transact(() => {
				sharedTeamIndexes.set(startTeam.team, newStartIndex);
				sharedTeamIndexes.set(moving!.team, newIndex);
				sharedTeamIndexes.set(endTeam.team, newEndIndex);
			});
		} else {
			sharedTeamIndexes.set(moving.team, generateJitteredKeyBetween(startIndex, endIndex));
		}

		moving = undefined;
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
				{@const thisMoving = moving?.team == row.team}
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
								moving = { team: row.team, index: row.index };
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
						{:else if moving.index > row.index}
							&lsh;
						{:else if moving.index < row.index}
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
