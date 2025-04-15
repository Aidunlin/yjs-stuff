<script lang="ts">
	import { onMount } from 'svelte';
	import {
		dndzone,
		overrideItemIdKeyNameBeforeInitialisingDndZones,
		SHADOW_ITEM_MARKER_PROPERTY_NAME,
		type DndEvent
	} from 'svelte-dnd-action';
	import { flip } from 'svelte/animate';
	import { IndexeddbPersistence } from 'y-indexeddb';
	import { WebrtcProvider } from 'y-webrtc';
	import * as Y from 'yjs';

	const SIGNAL_SERVER = 'https://y-webrtc-4ou5.onrender.com';
	const DOC_ID = 'my-document-id';
	const ARRAY_NAME = 'my-array';
	const MAP_NAME = 'my-map';

	const flipDurationMs = 100;

	type RankItem = { rank: number; team: string };

	let doc: Y.Doc;
	let sharedRanks: Y.Array<RankItem>;
	let sharedTeamState: Y.Map<true>;

	let undoManager = $state<Y.UndoManager>();
	let canUndo = $state(false);
	let canRedo = $state(false);

	let idb: IndexeddbPersistence;
	let webrtc: WebrtcProvider;

	let ranksView = $state<RankItem[]>([]);
	let teamStateView = $state(new Set());

	let activeTeams = $derived(ranksView.filter((r) => !teamStateView.has(r.team)));
	let crossedOffTeams = $derived(ranksView.filter((r) => teamStateView.has(r.team)));

	let draggedTeams = $derived(activeTeams);

	let alive = $state(!!localStorage.getItem('alive'));

	onMount(() => {
		overrideItemIdKeyNameBeforeInitialisingDndZones('team');
		alive && start();
	});

	function start() {
		doc = new Y.Doc();
		sharedRanks = doc.getArray(ARRAY_NAME);
		sharedTeamState = doc.getMap(MAP_NAME);

		idb = new IndexeddbPersistence(DOC_ID, doc);
		webrtc = new WebrtcProvider(DOC_ID, doc, { signaling: [SIGNAL_SERVER] });

		// Undoing/redoing moving items has issues in shared array.
		// Only do shared crossed-off state for now.
		// Possible fix: fractional indexing with randomized fraction offset.
		undoManager = new Y.UndoManager(sharedTeamState);
		undoManager.on('stack-item-added', updateCanUndo);
		undoManager.on('stack-item-popped', updateCanUndo);
		undoManager.on('stack-cleared', updateCanUndo);

		alive = true;
		localStorage.setItem('alive', 'true');

		sharedRanks.observe(() => {
			ranksView = sharedRanks.toArray();
		});

		sharedTeamState.observe(() => {
			teamStateView = new Set([...sharedTeamState.keys()]);
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
		ranksView = [];
	}

	function clear() {
		stop();
		idb.clearData();
	}

	function fill() {
		const defaultRanks: RankItem[] = [
			{ rank: 1, team: '2910' },
			{ rank: 2, team: '1778' },
			{ rank: 3, team: '2930' },
			{ rank: 4, team: '9450' },
			{ rank: 5, team: '3663' },
			{ rank: 6, team: '2046' },
			{ rank: 7, team: '9442' },
			{ rank: 8, team: '1540' }
		];

		const filtered = defaultRanks.filter((r) => !ranksView.some((v) => v.team != r.team));

		if (filtered.length) {
			sharedRanks.push(filtered);
		}
	}

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

	function correctDndItems(items: RankItem[], team: string) {
		if (items.filter((i) => i.team == team).length > 1) {
			return items.filter(
				(i: any) =>
					!teamStateView.has(i.team) && (i.team !== team || i[SHADOW_ITEM_MARKER_PROPERTY_NAME])
			);
		}

		return items.filter((i) => !teamStateView.has(i.team));
	}

	function onconsider({ detail }: CustomEvent<DndEvent<RankItem>>) {
		draggedTeams = correctDndItems(detail.items, detail.info.id);
	}

	function onfinalize({ detail }: CustomEvent<DndEvent<RankItem>>) {
		const team = detail.info.id;

		detail.items = correctDndItems(detail.items, team);

		const oldActualIndex = ranksView.findIndex((r) => r.team == team);
		const oldFilteredIndex = activeTeams.findIndex((r) => r.team == team);

		const diff = detail.items.findIndex((r) => r.team == team) - oldFilteredIndex;

		const newFilteredIndex = oldFilteredIndex + diff;
		const newActualIndex = ranksView.findIndex((r) => r.team == activeTeams[newFilteredIndex].team);

		if (newActualIndex !== oldActualIndex) {
			doc.transact(() => {
				const temp = sharedRanks.get(oldActualIndex);
				sharedRanks.delete(oldActualIndex);
				sharedRanks.insert(newActualIndex, [temp]);
			});
		}

		activeTeams = detail.items;
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
		<button onclick={fill}>fill</button>
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
			{#each draggedTeams as { rank, team }, index (team)}
				<div
					animate:flip={{ duration: flipDurationMs }}
					style="background-color:lightgray;width:fit-content;margin-block:4px;padding:2px"
				>
					<span style="display:inline-block;padding-left:6px">&UpDownArrow;</span>
					<input
						type="checkbox"
						style="width:20px;margin:4px"
						onchange={() => sharedTeamState.set(team, true)}
					/>
					<span style="display:inline-block;width:40px">{rank}{getOrdinal(Number(rank))}</span>
					<span style="display:inline-block;width:40px">{team}</span>
					<button onclick={() => move(team, -1)} disabled={index == 0}>&uarr;</button>
					<button onclick={() => move(team, 1)} disabled={index >= activeTeams.length - 1}>
						&darr;
					</button>
				</div>
			{/each}
		</div>
	{/if}

	{#if crossedOffTeams.length}
		<p>Crossed off</p>

		{#each crossedOffTeams as { rank, team } (team)}
			<div style="width:fit-content;margin-block:4px;padding:2px">
				<input
					type="checkbox"
					style="width:20px;margin:4px"
					checked
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
