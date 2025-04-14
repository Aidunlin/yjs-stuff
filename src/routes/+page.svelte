<script lang="ts">
	import { onMount } from 'svelte';
	import {
		dndzone,
		overrideItemIdKeyNameBeforeInitialisingDndZones,
		type DndEvent
	} from 'svelte-dnd-action';
	import { flip } from 'svelte/animate';
	import { IndexeddbPersistence } from 'y-indexeddb';
	import { WebrtcProvider } from 'y-webrtc';
	import * as Y from 'yjs';

	const SIGNAL_SERVER = 'http://localhost:4444';
	const DOC_ID = 'my-document-id';
	const ARRAY_NAME = 'my-array';
	const MAP_NAME = 'my-map';

	const flipDurationMs = 100;

	type RankItem = { rank: string; team: string };

	let doc: Y.Doc;
	let sharedRanks: Y.Array<RankItem>;
	let sharedTeamState: Y.Map<true>;

	let undoManager = $state<Y.UndoManager>();
	let canUndo = $state(false);
	let canRedo = $state(false);

	let idb: IndexeddbPersistence;
	let webrtc: WebrtcProvider;

	let ranksView = $state<RankItem[]>([]);
	let teamStateView = $state<string[]>([]);

	let activeTeams = $derived(ranksView.filter((r) => !teamStateView.includes(r.team)));
	let crossedOffTeams = $derived(ranksView.filter((r) => teamStateView.includes(r.team)));

	let draggedTeams = $derived(activeTeams);

	let inputs = $state({ rank: '', team: '' });

	let alive = $state(!!localStorage.getItem('alive'));

	onMount(() => {
		overrideItemIdKeyNameBeforeInitialisingDndZones('team');
		alive && start();
	});

	function start() {
		doc = new Y.Doc();
		sharedRanks = doc.getArray(ARRAY_NAME);
		sharedTeamState = doc.getMap(MAP_NAME);

		undoManager = new Y.UndoManager(doc);
		canUndo = undoManager.canUndo();
		canRedo = undoManager.canRedo();

		idb = new IndexeddbPersistence(DOC_ID, doc);
		webrtc = new WebrtcProvider(DOC_ID, doc, { signaling: [SIGNAL_SERVER] });

		alive = true;
		localStorage.setItem('alive', 'true');

		sharedRanks.observe(() => {
			ranksView = sharedRanks.toArray();
		});

		sharedTeamState.observe(() => {
			teamStateView = [...sharedTeamState.keys()];
		});

		undoManager.on('stack-item-added', updateCanUndo);
		undoManager.on('stack-item-popped', updateCanUndo);
		undoManager.on('stack-cleared', updateCanUndo);
	}

	function updateCanUndo() {
		canUndo = undoManager?.canUndo() ?? false;
		canRedo = undoManager?.canRedo() ?? false;
	}

	function stop() {
		alive = false;
		localStorage.removeItem('alive');

		doc.destroy();
		ranksView = [];
	}

	function clear() {
		stop();
		idb.clearData();
	}

	/*
	function move(team: string, by: number) {
		const oldActualIndex = ranksView.findIndex((r) => r.team == team);
		const oldFilteredIndex = activeTeams.findIndex((r) => r.team == team);

		const newFilteredIndex = oldFilteredIndex + by;
		const newActualIndex = ranksView.findIndex((r) => r.team == activeTeams[newFilteredIndex].team);

		doc.transact(() => {
			const temp = sharedRanks.get(oldActualIndex);
			sharedRanks.delete(oldActualIndex);
			sharedRanks.insert(newActualIndex, [temp]);
		});
	}
	*/

	function onconsider(e: CustomEvent<DndEvent<RankItem>>) {
		draggedTeams = e.detail.items;
	}

	function onfinalize(e: CustomEvent<DndEvent<RankItem>>) {
		const team = e.detail.info.id;

		const oldActualIndex = ranksView.findIndex((r) => r.team == team);
		const oldFilteredIndex = activeTeams.findIndex((r) => r.team == team);

		const diff = e.detail.items.findIndex((r) => r.team == team) - oldFilteredIndex;

		const newFilteredIndex = oldFilteredIndex + diff;
		const newActualIndex = ranksView.findIndex((r) => r.team == activeTeams[newFilteredIndex].team);

		if (newActualIndex !== oldActualIndex) {
			doc.transact(() => {
				const temp = sharedRanks.get(oldActualIndex);
				sharedRanks.delete(oldActualIndex);
				sharedRanks.insert(newActualIndex, [temp]);
			});
		}

		activeTeams = e.detail.items;
	}

	function getOrdinal(n: number) {
		if (n % 10 == 1 && n % 100 != 11) return 'st';
		if (n % 10 == 2 && n % 100 != 12) return 'nd';
		if (n % 10 == 3 && n % 100 != 13) return 'rd';
		return 'th';
	}
</script>

<h1>yjs-stuff</h1>

{#if alive}
	<p>
		<button onclick={stop}>stop</button>
		<button onclick={clear}>clear and stop</button>
	</p>

	<p>
		<input bind:value={inputs.rank} placeholder="rank" />
		<input bind:value={inputs.team} placeholder="team" />

		<button
			onclick={() => {
				sharedRanks.push([inputs]);
				inputs = { rank: '', team: '' };
			}}
		>
			add item
		</button>
	</p>

	<p>
		<button onclick={() => undoManager?.undo()} disabled={!canUndo}>undo</button>
		<button onclick={() => undoManager?.redo()} disabled={!canRedo}>redo</button>
	</p>

	{#if activeTeams.length}
		<p>Teams</p>

		<div
			use:dndzone={{ items: $state.snapshot(draggedTeams), flipDurationMs }}
			{onconsider}
			{onfinalize}
		>
			{#each draggedTeams as { rank, team } (team)}
				<div
					animate:flip={{ duration: flipDurationMs }}
					style="background-color:lightgray;width:fit-content;margin-block:4px;padding:2px"
				>
					<span style="display:inline-block;padding-left:6px">&UpDownArrow;</span>
					<input
						type="checkbox"
						style="width:20px;margin:4px"
						checked={teamStateView.includes(team)}
						onchange={() => sharedTeamState.set(team, true)}
					/>
					<span style="display:inline-block;width:40px">{rank}{getOrdinal(Number(rank))}</span>
					<span style="display:inline-block;width:40px">{team}</span>
					<!--
					<button onclick={() => move(team, -1)} disabled={index == 0}>&uarr;</button>
					<button onclick={() => move(team, 1)} disabled={index >= activeTeams.length - 1}>
						&darr;
					</button>
					-->
				</div>
			{/each}
		</div>
	{/if}

	{#if crossedOffTeams.length}
		<p>Crossed off</p>

		{#each crossedOffTeams as { rank, team } (team)}
			<div style="background-color:lightgray;width:fit-content;margin-block:4px;padding:2px">
				<input
					type="checkbox"
					style="width:20px;margin:4px"
					checked={teamStateView.includes(team)}
					onchange={() => sharedTeamState.delete(team)}
				/>
				<span style="display:inline-block;width:40px;text-decoration-line:line-through">
					{rank}{getOrdinal(Number(rank))}
				</span>
				<span style="display:inline-block;width:40px;text-decoration-line:line-through">
					{team}
				</span>
			</div>
		{/each}
	{/if}
{:else}
	<p>
		<button onclick={start}>start</button>
	</p>
{/if}
