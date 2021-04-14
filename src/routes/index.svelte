<script>
	import { onMount } from 'svelte';
	import { scale } from 'svelte/transition';
	import { quintOut } from 'svelte/easing';

	function adjustColor(color, percent) {
		let R = parseInt(color.substring(1, 3), 16);
		let G = parseInt(color.substring(3, 5), 16);
		let B = parseInt(color.substring(5, 7), 16);

		R = parseInt((R * (100 + percent)) / 100);
		G = parseInt((G * (100 + percent)) / 100);
		B = parseInt((B * (100 + percent)) / 100);

		R = R < 255 ? R : 255;
		G = G < 255 ? G : 255;
		B = B < 255 ? B : 255;

		let RR = R.toString(16).length === 1 ? '0' + R.toString(16) : R.toString(16);
		let GG = G.toString(16).length === 1 ? '0' + G.toString(16) : G.toString(16);
		let BB = B.toString(16).length === 1 ? '0' + B.toString(16) : B.toString(16);

		return '#' + RR + GG + BB;
	}

	const ALGORITHMS = {
		BINARY_SEARCH: 'BINARY_SEARCH',
		LINEAR_SEARCH: 'LINEAR_SEARCH',
		JUMP_SEARCH: 'JUMP_SEARCH',
		INTERPOLATION_SEARCH: 'INTERPOLATION_SEARCH',

		BUBBLE_SORT: 'BUBBLE_SORT'
	};

	let executing = false;
	let matrix = [];
	let isMatrixSorted = true;
	let pointToFind = { row: null, column: null, value: null };
	let pointToStart = { row: null, column: null, value: null };
	let dragging = false;

	let isDraggingPointToFind = false;
	let ROWS = ((window.innerHeight - 60) / 40) >> 0;
	let COLUMNS = (window.innerWidth / 40) >> 0;
	let MAX = ROWS > COLUMNS ? COLUMNS : ROWS;

	const randomIntFromInterval = (min, max) => {
		return Math.floor(Math.random() * (max - min + 1) + min);
	};

	const initializeMatrix = () => {
		isMatrixSorted = true;
		let value = 1;
		for (let i = 0; i < MAX; i++) {
			if (!matrix[i]) matrix[i] = [];
			for (let j = 0; j < MAX; j++) {
				matrix[i][j] = { value, selected: false };
				// value += 2 + randomIntFromInterval(0, 10);
				value++;
			}
		}
	};

	onMount(() => {
		initializeMatrix();
	});

	let selectedAlgo = ALGORITHMS.LINEAR_SEARCH;
	let stepsToFind = 0;

	const shuffleMatrix = () => {
		isMatrixSorted = false;
		let temporaryValue;
		for (const row in matrix) {
			for (const column in matrix[row]) {
				const rand1 = randomIntFromInterval(0, matrix.length - 1);
				const rand2 = randomIntFromInterval(0, matrix.length - 1);
				temporaryValue = matrix[row][column];
				matrix[row][column] = matrix[rand1][rand2];
				matrix[rand1][rand2] = temporaryValue;
			}
		}
	};

	const resetSelectedPoints = () => {
		return matrix.forEach((v, i) =>
			v.forEach((v1, j) => {
				matrix[i][j].selected = false;
			})
		);
	};

	const getMatrixAsArray = () => {
		const arr = [];

		for (const row in matrix) {
			for (const column in matrix[row]) {
				arr.push({ value: matrix[row][column].value, row, column });
			}
		}

		return arr;
	};

	const hasFoundPoint = () => {
		let found = false;

		if (matrix[pointToFind.row][pointToFind.column].selected) {
			found = true;
			setTimeout(() => {
				alert(`Found all points in ${stepsToFind} steps`);
			}, 0);
		}

		return found;
	};

	const interpolationSearch = async (lo, hi) => {
		let arr = getMatrixAsArray();
		const x = pointToFind.value;

		let pos;

		// Since array is sorted, an element present
		// in array must be in range defined by corner

		if (lo <= hi && x >= arr[lo].value && x <= arr[hi].value) {
			// Probing the position with keeping
			// uniform distribution in mind.
			pos = lo + Math.floor(((hi - lo) / (arr[hi].value - arr[lo].value)) * (x - arr[lo].value));

			matrix[arr[pos].row][arr[pos].column].selected = true;
			stepsToFind++;

			// Condition of target found
			if (arr[pos].value === x) {
				return hasFoundPoint();
			}

			await new Promise((r) => setTimeout(r, 500));

			// If x is larger, x is in right sub array
			if (arr[pos].value < x) {
				return interpolationSearch(pos + 1, hi);
			}

			// If x is smaller, x is in left sub array
			if (arr[pos].value > x) {
				return interpolationSearch(lo, pos - 1);
			}
		}
	};

	const jumpSearch = async () => {
		const arr = getMatrixAsArray();

		const s = pointToFind.value;

		// Finding block size to be jumped
		let step = Math.sqrt(arr.length);

		// Finding the block where element is
		// present (if it is present)
		let prev = 0;
		while (arr[Math.min(step, arr.length) - 1].value < s) {
			prev = step;
			step += Math.sqrt(arr.length);
			matrix[arr[Math.min(step, arr.length) - 1].row][
				arr[Math.min(step, arr.length) - 1].column
			].selected = true;
			stepsToFind++;
			await new Promise((r) => setTimeout(r, 0));
			if (prev >= arr.length) return;
		}

		// Doing a linear search for x in block
		// beginning with prev.
		while (arr[prev].value < s) {
			prev++;
			matrix[arr[prev].row][arr[prev].column].selected = true;
			stepsToFind++;
			await new Promise((r) => setTimeout(r, 0));

			// If we reached next block or end of
			// array, element is not present.
			if (prev == Math.min(step, arr.length)) return;
		}
		// If element is found
		if (arr[prev].value == s) return hasFoundPoint();
	};

	const binarySearch = async () => {
		const arr = getMatrixAsArray();

		let start = 0;
		let end = arr.length - 1;

		while (start <= end) {
			const middle = Math.floor((start + end) / 2);
			matrix[arr[middle].row][arr[middle].column].selected = true;

			if (arr[middle].value === pointToFind.value) {
				return hasFoundPoint();
			} else if (arr[middle].value < pointToFind.value) {
				// continue searching to the right
				start = middle + 1;
			} else {
				// search searching to the left
				end = middle - 1;
			}
			stepsToFind++;
			await new Promise((r) => setTimeout(r, 300));
		}
	};

	const linearSearch = async () => {
		const arr = getMatrixAsArray();

		let i = 0;
		for (const elem of arr) {
			matrix[elem.row][elem.column].selected = true;
			stepsToFind++;
			if (hasFoundPoint()) {
				return;
			}
			if (i % 3 === 0) {
				await new Promise((r) => setTimeout(r, 0));
			}
			i++;
		}
	};

	const bubbleSort = async () => {
		const arr = getMatrixAsArray();

		for (let i = arr.length - 1; i >= 0; i--) {
			let shouldShow = false;
			for (let j = 1; j <= i; j++) {
				if (arr[j - 1].value > arr[j].value) {
					shouldShow = true;
					// Update array
					const prev = { ...arr[j - 1] };
					const next = { ...arr[j] };

					arr[j].row = prev.row;
					arr[j].column = prev.column;
					arr[j - 1].row = next.row;
					arr[j - 1].column = next.column;

					const objPrev = { ...matrix[prev.row][prev.column] };
					const objNext = { ...matrix[next.row][next.column] };

					matrix[prev.row][prev.column] = objNext;
					matrix[next.row][next.column] = objPrev;

					const temp = arr[j - 1];
					arr[j - 1] = arr[j];
					arr[j] = temp;
				}
			}
			if (shouldShow) {
				shouldShow = false;
				await new Promise((r) => setTimeout(r, 0));
			}
		}
	};

	const startWork = async () => {
		executing = true;
		if (!isMatrixSorted && selectedAlgo !== ALGORITHMS.BUBBLE_SORT) {
			initializeMatrix();
		}

		if (!pointToFind.value && selectedAlgo !== ALGORITHMS.BUBBLE_SORT) {
			generateRandomPointToFind();
		}

		resetSelectedPoints();
		stepsToFind = 0;
		switch (selectedAlgo) {
			case ALGORITHMS.BUBBLE_SORT:
				await bubbleSort();
				break;
			case ALGORITHMS.JUMP_SEARCH:
				await jumpSearch();
				break;
			case ALGORITHMS.LINEAR_SEARCH:
				await linearSearch();
				break;
			case ALGORITHMS.INTERPOLATION_SEARCH:
				await interpolationSearch(0, getMatrixAsArray().length - 1);
				break;
			case ALGORITHMS.BINARY_SEARCH:
			default:
				await binarySearch();
				break;
		}
		executing = false;
	};

	const generateRandomPointToFind = () => {
		const arr = getMatrixAsArray();
		let index = randomIntFromInterval(0, arr.length - 1);
		pointToFind = {
			row: arr[index].row >> 0,
			column: arr[index].column >> 0,
			value: arr[index].value >> 0
		};
	};
</script>

<svelte:head>
	<title>Hello world!</title>
</svelte:head>

<main>
	<div class="header">
		<button
			on:click={() => {
				shuffleMatrix();
			}}
		>
			Shuffle
		</button>
		<button
			on:click={() => {
				resetSelectedPoints();
				pointToFind = { row: null, column: null, value: null };
				pointToStart = { row: null, column: null, value: null };
			}}
		>
			Reset
		</button>
		<button on:click={generateRandomPointToFind}> Add/Change point to find </button>
		<button on:click={startWork} disabled={executing || undefined}>Execute</button>
		<select on:change={(e) => (selectedAlgo = e.target.value)}>
			<option value={ALGORITHMS.LINEAR_SEARCH}>Linear Search</option>
			<option value={ALGORITHMS.BINARY_SEARCH}>Binary Search</option>
			<option value={ALGORITHMS.JUMP_SEARCH}>Jump Search</option>
			<option value={ALGORITHMS.INTERPOLATION_SEARCH}>Interpolation Search</option>
			<option value={ALGORITHMS.BUBBLE_SORT}>Bubble Sort</option>
		</select>
	</div>
	<table on:contextmenu={(e) => e.preventDefault()}>
		{#each matrix as m, i}
			<tr>
				{#each m as m1, j}
					<th
						on:mousedown={(e) => {
							e.preventDefault();
							if (i === pointToFind.row && j === pointToFind.column) {
								isDraggingPointToFind = true;
								return;
							}

							dragging = e.button;
							if (dragging === 0) {
								matrix[i][j].selected = true;
							} else {
								matrix[i][j].selected = false;
							}
						}}
						on:mouseover={() => {
							if (isDraggingPointToFind) {
								pointToFind = { row: i, column: j, value: matrix[i][j] };
								return;
							}
							if (dragging !== false) {
								if (dragging === 0) {
									matrix[i][j].selected = true;
								} else {
									matrix[i][j].selected = false;
								}
							}
						}}
						on:mouseup={() => {
							if (isDraggingPointToFind) {
								isDraggingPointToFind = false;
								return;
							}
							dragging = false;
						}}
						style={`background: ${adjustColor('#ffffff', -(matrix[i][j].value / 11))}`}
					>
						{#if matrix[i][j].selected || (pointToFind.row === i && pointToFind.column === j) || (pointToStart.row === i && pointToStart.column === j)}
							<div
								class="thInner"
								class:selected={matrix[i][j].selected}
								class:toFind={pointToFind.row === i && pointToFind.column === j}
								class:toStart={pointToStart.row === i && pointToStart.column === j}
								transition:scale={{
									duration: isDraggingPointToFind || executing ? 0 : 200,
									delay: 0,
									start: isDraggingPointToFind && executing ? 1 : 0,
									easing: quintOut
								}}
								style={`cursor: ${
									pointToFind.row === i && pointToFind.column === j && !executing
										? 'move'
										: 'default'
								}`}
							/>
						{/if}
					</th>
				{/each}
			</tr>
		{/each}
	</table>
</main>

<svelte:body
	on:mouseleave={() => {
		dragging = false;
		isDraggingPointToFind = false;
	}} />

<style>
	.header {
		display: flex;
		height: 60px;
	}

	main {
		text-align: center;
		-webkit-user-select: none;
		display: grid;
		place-items: center;
	}
	table {
		border-spacing: 0;
	}
	th {
		box-sizing: border-box;
		border: 1px solid black;
		width: 40px;
		height: 40px;
		padding: 0;
		margin: 0;
		position: relative;
	}
	.thInner {
		display: block;
		width: 40px;
		height: 40px;
		position: absolute;
		margin: 0;
		top: 0;
	}
	.selected {
		background: black;
	}
	.toFind {
		background: red;
	}
	.toStart {
		background: purple;
	}
</style>
