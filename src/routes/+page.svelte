<!-- svelte-ignore a11y-click-events-have-key-events -->
<!-- svelte-ignore a11y-no-static-element-interactions -->
<svg viewBox="0 0 100 100"
	on:click={evt => {
		if (moveMode) return;

		let x2 = (evt.offsetX / evt.currentTarget.clientWidth) * 100;
		let y2 = (evt.offsetY / evt.currentTarget.clientHeight) * 100;

		if (snapMode) {
			x2 = Math.round(x2);
			y2 = Math.round(y2);
		}

		const dist = 2;
		for (const [k, {x: x1, y: y1}] of Object.entries(embedding)) {
			if (
				(x1-dist <= x2 && x2 <= x1+dist) &&
				(y1-dist <= y2 && y2 <= y1+dist)
			) {
				return;
			}
		}
		addVertex(x2, y2);
	}}

	on:pointermove={evt => {
		if (drag_from != null) {
			let x = (evt.offsetX / evt.currentTarget.clientWidth) * 100;
			let y = (evt.offsetY / evt.currentTarget.clientHeight) * 100;

			if (snapMode) {
				x = Math.round(x);
				y = Math.round(y);
			}

			if (moveMode) {
				embedding[drag_from] = {x, y};
			} else {
				drag_to = {x, y};
			}
		}
	}}

	on:pointerup={evt => {
		drag_from = null;
		drag_to = null;
	}}
>
	{#each Array.from({length: 100}, (_, i) => i) as i}
		{#each Array.from({length: 100}, (_, j) => j) as j}
			<circle
				r="0.1"
				cx={i}
				cy={j}
				fill="lightgray"
			/>
		{/each}
	{/each}
	{#if drag_from != null && drag_to != null}
		<line
			stroke="black"
			x1={embedding[drag_from].x}
			y1={embedding[drag_from].y}
			x2={drag_to.x}
			y2={drag_to.y}
		/>
	{/if}
	<g id="edges_container">
		{#each Object.entries(adj) as [k, v]}
			{#each v as u}
				<line
					on:click={(evt) => {
						evt.stopPropagation();
						removeEdge(Number(k), Number(u));
					}}
					x1={embedding[Number(k)].x}
					y1={embedding[Number(k)].y}
					x2={embedding[u].x}
					y2={embedding[u].y}
				/>
			{/each}
		{/each}
	</g>
	<g id="vertices_container">
		{#each Object.entries(adj) as [k, v]}
			<circle
				style="pointer-events: none;"
				r="1.4"
				cx={embedding[Number(k)].x}
				cy={embedding[Number(k)].y}
				fill="white"
			/>
			<circle
				r="1"
				on:click={(evt) => {
					if (moveMode) return;
					evt.stopPropagation();
					removeVertex(Number(k));
				}}
				on:pointerdown={(evt) => {
					evt.stopPropagation();
					drag_from = Number(k)
				}}
				on:pointerup={(evt) => {
					if (moveMode) return;
					if (drag_from) {
						addEdge(drag_from, Number(k));
					}
				}}
				cx={embedding[Number(k)].x}
				cy={embedding[Number(k)].y}
			/>
			{#if textMode}
				<text
					class="unselectable"
					x={embedding[Number(k)].x}
					y={embedding[Number(k)].y}
					font-size="0.3em"
					style="pointer-events: none; user-select: none;"
					fill="red"
				>{k}</text>
			{/if}
		{/each}
	</g>
</svg>
<br>
<input type="checkbox" name="move" id="" bind:checked={moveMode}>
<label for="move">{moveMode ? "move mode" : "edit mode"}</label>
<input type="checkbox" name="snap" id="" bind:checked={snapMode}>
<label for="snap">{snapMode ? "snap to grid" : "no snap"}</label>
<input type="checkbox" name="text" id="" bind:checked={textMode}>
<label for="text">{textMode ? "show text" : "hide text"}</label>
<p
	contenteditable
	bind:innerText={error_msg}
	style="color: red"
></p>
<textarea
	on:input={evt => updateGraph(evt.currentTarget.value)}
>{data_string}</textarea>

<script lang="ts">
	$: moveMode = false;
	$: snapMode = true;
	$: textMode = true;

	let N = 0;
	let NextId = N;

	let adj: Record<number, Set<number>> = {}

	let embedding: Record<number, {x: number, y: number}> = {}

	function addVertex(x: number, y: number) {
		N++;
		const newId = NextId++;
		console.log("add", newId)
		
		adj[newId] = new Set();
		embedding[newId] = {x, y};

		adj = adj; // force reaction
	}

	function removeVertex(id: number) {
		console.log("remove", id)
		
		N--;
		
		delete adj[id];
		delete embedding[id];
		
		for (const [k, v] of Object.entries(adj)) {
			v.delete(id);
		}

		adj = adj; // force reaction
	}

	function addEdge(u: number, v: number) {
		console.log("add", u, "<-->", v)
		adj[u].add(v);
		adj[v].add(u);
		adj = adj; // force reaction
	}
	
	function removeEdge(u: number, v: number) {
		console.log("remove", u, "<-->", v)
		adj[u].delete(v);
		adj[v].delete(u);
		adj = adj; // force reaction
	}

	$: data_string = String(N) + '\n' +
		Object.entries(embedding)
			.reduce((prev, curr, i) => {
				return prev += `${curr[0]} ${curr[1].x} ${curr[1].y}\n`
			}, "")
		+ Object.entries(adj)
			.reduce((prev, curr, i) => {
				return prev += `${curr[0]} ${Array.from(curr[1]).join(' ')}\n`
			}, "").trim()

	let drag_from: number | null = null;

	let drag_to: {x: number, y: number} | null = null;

	let error_msg = ''

	function updateGraph(data: string) {
		try {
			let lines = data.split('\n')

			const newN = Number(lines[0])

			if (isNaN(newN) || !Number.isInteger(newN)) {
				throw new Error(`Expected an integer, found: ${lines[0]}`);
			}

			lines = lines.splice(1);

			if (lines.length != 2 * newN) {
				throw new Error(`Expected ${2*newN} lines, found: ${lines.length}`);
			}

			const embedding_lines = lines.slice(0, newN);

			const newEmbedding: Record<number, {x: number, y: number}> = {}

			for (let i = 0; i < newN; i++) {
				const tokens = embedding_lines[i].trim().split(' ')
				if (tokens.length != 3)
					throw new Error(`Expected 3 tokens on this line, found ${tokens.length}: ${tokens}`);

				const v = Number(tokens[0]);
				if (isNaN(v) || !Number.isInteger(v))
					throw new Error(`Expected an integer, found: ${tokens[0]}`);
				
				const x = Number(tokens[1]);
				if (isNaN(x))
					throw new Error(`Expected a number, found: ${tokens[1]}`);

				const y = Number(tokens[2]);
				if (isNaN(y))
					throw new Error(`Expected a number, found: ${tokens[2]}`);

				newEmbedding[v] = {x, y};
			}

			const adj_lines = lines.slice(newN);

			const newAdj: Record<number, Set<number>> = {}
			for (let i = 0; i < newN; i++) {
				const tokens = adj_lines[i].trim().split(' ')
				if (tokens.length < 1)
					throw new Error(`Expected 1 token on this line, found ${tokens.length}: ${tokens}`);

				const v = Number(tokens[0]);
				if (isNaN(v) || !Number.isInteger(v))
					throw new Error(`Expected an integer, found: ${tokens[0]}`);
				
				const neighbors = new Set<number>();

				for (let j = 1; j < tokens.length; j++) {
					const u = Number(tokens[j]);
					if (isNaN(u) || !Number.isInteger(u))
						throw new Error(`Expected an integer, found: ${tokens[j]}`);
					if (Object.keys(newEmbedding).find((v) => Number(v) == u) == undefined)
						throw new Error(`Expected an already defined vertex, found: ${tokens[j]}`);
					neighbors.add(u)
				}

				newAdj[v] = neighbors;
			}

			N = newN;
			NextId = Object.keys(newEmbedding).reduce((prev, curr) => Math.max(prev, Number(curr)), 0) + 1;
			console.log("NextId", NextId)
			embedding = newEmbedding;
			adj = newAdj;
			error_msg = ""
		} catch (e) {
			if (
				typeof e === "object" &&
				e &&
				"message" in e &&
				typeof e.message === "string"
			) {
				error_msg = e.message
			}
		}
	}

	function onKeyDown(evt: KeyboardEvent) {
		if (evt.key === "Shift") {
			moveMode = true;
		}
	}
	function onKeyUp(evt: KeyboardEvent) {
		if (evt.key === "Shift") {
			moveMode = false;
		}
	}
</script>

<svelte:window
	on:keydown={onKeyDown}
	on:keyup={onKeyUp}
/>

<style>
	svg {
		border: 1px solid black;
	}

	textarea {
		width: 100%; min-height: 400px;
	}

	line {
		stroke: black;
	}

	.unselectable {
		-webkit-touch-callout: none;
		-webkit-user-select: none;
		-khtml-user-select: none;
		-moz-user-select: none;
		-ms-user-select: none;
		user-select: none;
	}
</style>