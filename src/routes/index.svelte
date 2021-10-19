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

	const SECTIONS = {
		SORT: 'SORT',
		SEARCH: 'SEARCH',
		MAZE: 'MAZE'
	};

	const ALGORITHMS = {
		LINEAR_SEARCH: {
			id: 'LINEAR_SEARCH',
			needSorting: false,
			name: 'Linear Search',
			section: SECTIONS.SEARCH
		},
		BINARY_SEARCH: {
			id: 'BINARY_SEARCH',
			needSorting: true,
			name: 'Binary Search',
			section: SECTIONS.SEARCH
		},
		JUMP_SEARCH: {
			id: 'JUMP_SEARCH',
			needSorting: true,
			name: 'Jump Search',
			section: SECTIONS.SEARCH
		},
		INTERPOLATION_SEARCH: {
			id: 'INTERPOLATION_SEARCH',
			needSorting: true,
			name: 'Interpolation Search',
			section: SECTIONS.SEARCH
		},

		BUBBLE_SORT: {
			id: 'BUBBLE_SORT',
			needSorting: true,
			name: 'Bubble Sort',
			section: SECTIONS.SORT
		},
		SELECTION_SORT: {
			id: 'SELECTION_SORT',
			needSorting: true,
			name: 'Selection Sort',
			section: SECTIONS.SORT
		},
		INSERTION_SORT: {
			id: 'INSERTION_SORT',
			needSorting: true,
			name: 'Insertion Sort',
			section: SECTIONS.SORT
		}
		// MERGE_SORT: {
		// 	id: 'MERGE_SORT',
		// 	needSorting: true,
		// 	name: 'Merge Sort',
		// 	section: SECTIONS.SORT
		// }
	};

	let ALGORITHM_SECTIONS = {};

	for (const algo in ALGORITHMS) {
		const algorithm = ALGORITHMS[algo];
		ALGORITHM_SECTIONS[algorithm.section] = (ALGORITHM_SECTIONS[algorithm.section] || []).concat(
			algorithm
		);
	}

	let executing = false;
	let matrix = [];
	let isMatrixSorted = true;
	let pointToFind = { row: null, column: null, value: null };
	let pointToStart = { row: null, column: null, value: null };
	let dragging = false;

	let isDraggingPointToFind = false;

	const randomIntFromInterval = (min, max) => {
		return Math.floor(Math.random() * (max - min + 1) + min);
	};

	let ROWS = 0;
	let COLUMNS = 0;

	const initializeMatrix = () => {
		isMatrixSorted = true;
		let value = 1;
		let id = 1;
		for (let i = 0; i < ROWS; i++) {
			if (!matrix[i]) matrix[i] = [];
			for (let j = 0; j < COLUMNS; j++) {
				matrix[i][j] = { value, selected: false, id };
				value += 2 + randomIntFromInterval(0, 10);
				id++;
			}
		}
	};

	onMount(() => {
		ROWS = ((window.innerHeight - 250) / 40) >> 0;
		COLUMNS = (window.innerWidth / 40) >> 0;
		initializeMatrix();
	});

	let selectedAlgo = ALGORITHMS.LINEAR_SEARCH.id;

	$: {
		if (ALGORITHMS[selectedAlgo].section === SECTIONS.SORT) {
			shuffleMatrix();
			resetSelectedPoints();
			pointToFind = { row: null, column: null, value: null };
		} else {
			initializeMatrix();
		}
	}

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
			v.forEach((_, j) => {
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

	const swapMatrixValues = (arr, a, b) => {
		const prev = { ...arr[a] };
		const next = { ...arr[b] };

		arr[b].row = prev.row;
		arr[b].column = prev.column;
		arr[a].row = next.row;
		arr[a].column = next.column;

		const objPrev = { ...matrix[prev.row][prev.column] };
		const objNext = { ...matrix[next.row][next.column] };

		matrix[prev.row][prev.column] = objNext;
		matrix[next.row][next.column] = objPrev;
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
		let step = Math.sqrt(arr.length) >> 0;

		// Finding the block where element is
		// present (if it is present)
		let prev = 0;
		while (arr[Math.min(step, arr.length) - 1].value < s) {
			prev = step;
			step += Math.sqrt(arr.length) >> 0;
			matrix[arr[Math.min(step, arr.length) - 1].row][
				arr[Math.min(step, arr.length) - 1].column
			].selected = true;
			stepsToFind++;
			await new Promise((r) => setTimeout(r, 100));
			if (prev >= arr.length) return;
		}

		// Doing a linear search for x in block
		// beginning with prev.
		while (arr[prev].value < s) {
			prev++;
			matrix[arr[prev].row][arr[prev].column].selected = true;
			stepsToFind++;
			await new Promise((r) => setTimeout(r, 100));

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
			for (let j = 1; j <= i; j++) {
				if (arr[j - 1].value > arr[j].value) {
					// Update array
					swapMatrixValues(arr, j - 1, j);

					const temp = arr[j - 1];
					arr[j - 1] = arr[j];
					arr[j] = temp;

					if (j % (i / 2) === 0) {
						await new Promise((r) => setTimeout(r, 0));
					}
				}
			}
		}
	};

	const selectionSort = async () => {
		const arr = getMatrixAsArray();

		let minIdx;
		const len = arr.length;

		for (let i = 0; i < len; i++) {
			minIdx = i;
			for (let j = i + 1; j < len; j++) {
				if (arr[j].value < arr[minIdx].value) {
					minIdx = j;
				}
			}

			swapMatrixValues(arr, i, minIdx);

			const temp = arr[i];
			arr[i] = arr[minIdx];
			arr[minIdx] = temp;
			await new Promise((r) => setTimeout(r, 0));
		}
	};

	const insertionSort = async () => {
		const arr = getMatrixAsArray();
		let i, key, j;
		for (i = 1; i < arr.length; i++) {
			key = { ...arr[i] };
			const objKey = { ...matrix[arr[i].row][arr[i].column] };
			j = i - 1;

			while (j >= 0 && arr[j].value > key.value) {
				matrix[arr[j + 1].row][arr[j + 1].column] = { ...matrix[arr[j].row][arr[j].column] };
				arr[j + 1].value = arr[j].value;
				j = j - 1;
			}

			await new Promise((r) => setTimeout(r, 0));
			matrix[arr[j + 1].row][arr[j + 1].column] = objKey;
			arr[j + 1].value = key.value;
		}
	};

	const mergeSort = () => {
		const merge = (left, right) => {
			let arr = [];
			// Break out of loop if any one of the array gets empty
			while (left.length && right.length) {
				// Pick the smaller among the smallest element of left and right sub arrays
				if (left[0].value < right[0].value) {
					arr.push(left.shift());
				} else {
					let prev = { matrix: matrix[right[0].row][right[0].column], data: right[0] };
					console.log('moving', right[0], matrix[right[0].row][right[0].column]);

					const prev1 = { ...right[0] };
					const next1 = { ...left[left.length - 1] };

					left[left.length - 1].row = prev1.row;
					left[left.length - 1].column = prev1.column;
					right[0].row = next1.row;
					right[0].column = next1.column;

					const objPrev = { ...matrix[prev1.row][prev1.column] };
					const objNext = { ...matrix[next1.row][next1.column] };

					matrix[prev1.row][prev1.column] = objNext;
					matrix[next1.row][next1.column] = objPrev;

					for (let i = left.length - 2; i >= 0; i--) {
						const tmp = { ...left[i + 1] };

						const objPrev3 = { ...matrix[left[i].row][left[i].column] };

						left[i].row = left[i + 1].row;
						left[i].column = left[i + 1].column;

						matrix[tmp.row][tmp.column] = objPrev3;

						left[i + 1] = left[i];
					}

					const prev2 = { ...left[0] };

					right[0].row = prev2.row;
					right[0].column = prev2.column;
					left[0].row = prev.data.row;
					left[0].column = prev.data.column;

					const objPrev1 = { ...matrix[prev2.row][prev2.column] };

					matrix[prev2.row][prev2.column] = prev.matrix;
					matrix[prev.data.row][prev.data.column] = objPrev1;

					arr.push(right.shift());
				}
			}

			// Concatenating the leftover elements
			// (in case we didn't go through the entire left or right array)
			return [...arr, ...left, ...right];
		};
		const array = getMatrixAsArray();

		const runMerge = (arr) => {
			const half = arr.length / 2;
			// Base case or terminating case
			if (arr.length < 2) {
				return arr;
			}

			const left = arr.splice(0, half);
			return merge(runMerge(left), runMerge(arr));
		};

		const sorted = runMerge(array);
		// console.log(sorted);
	};

	const startWork = async () => {
		executing = true;
		if (ALGORITHMS[selectedAlgo].section === SECTIONS.SEARCH) {
			generateRandomPointToFind();
		} else {
			pointToFind = { row: null, column: null, value: null };
		}

		if (isMatrixSorted && ALGORITHMS[selectedAlgo].section === SECTIONS.SORT) {
			shuffleMatrix();
		}

		resetSelectedPoints();
		stepsToFind = 0;
		switch (selectedAlgo) {
			case ALGORITHMS.BUBBLE_SORT.id:
				await bubbleSort();
				break;
			case ALGORITHMS.SELECTION_SORT.id:
				await selectionSort();
				break;
			case ALGORITHMS.INSERTION_SORT.id:
				await insertionSort();
				break;
			// case ALGORITHMS.MERGE_SORT.id:
			// 	await mergeSort();
			// 	break;
			case ALGORITHMS.JUMP_SEARCH.id:
				await jumpSearch();
				break;
			case ALGORITHMS.LINEAR_SEARCH.id:
				await linearSearch();
				break;
			case ALGORITHMS.INTERPOLATION_SEARCH.id:
				await interpolationSearch(0, getMatrixAsArray().length - 1);
				break;
			case ALGORITHMS.BINARY_SEARCH.id:
			default:
				await binarySearch();
				break;
		}

		if (ALGORITHMS[selectedAlgo].section === SECTIONS.SORT) {
			isMatrixSorted = true;
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
		<h1>Algorithms Visualizer</h1>
		<div>
			<button on:click={startWork} disabled={executing || undefined}>Execute</button>
			<select bind:value={selectedAlgo} disabled={executing || undefined}>
				{#each Object.keys(ALGORITHM_SECTIONS) as sectionKey}
					<optgroup label={sectionKey}>
						{#each ALGORITHM_SECTIONS[sectionKey] as algorithm}
							<option value={algorithm.id}>{algorithm.name}</option>
						{/each}
					</optgroup>
				{/each}
			</select>
		</div>
		<div>
			Welcome to my algorithms visualizer. Please use your mouse to place some walls and then start
			the execution
		</div>
	</div>
	<table on:contextmenu={(e) => e.preventDefault()}>
		{#each matrix as m, i}
			<tr>
				{#each m as m1, j}
					<th
						on:mousedown={(e) => {
							e.preventDefault();
							if (executing) return;
							const buttonClicked = e.button;
							if (
								i === pointToFind.row &&
								j === pointToFind.column &&
								ALGORITHMS[selectedAlgo].section === SECTIONS.MAZE
							) {
								if (buttonClicked === 0) {
									isDraggingPointToFind = true;
								} else {
									pointToFind = { row: null, column: null, value: null };
								}
								return;
							}

							dragging = buttonClicked;
							if (dragging === 0) {
								matrix[i][j].selected = true;
							} else {
								matrix[i][j].selected = false;
							}
						}}
						on:mouseover|stopPropagation={() => {
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
						style="{ALGORITHMS[selectedAlgo].section === SECTIONS.SORT &&
							`background: ${adjustColor(
								'#ffffff',
								-(matrix[i][j].id / ((ROWS * COLUMNS) / 94))
							)};`}}"
					>
						{#if matrix[i][j].selected || (pointToFind.row === i && pointToFind.column === j) || (pointToStart.row === i && pointToStart.column === j)}
							<div
								class="thInner"
								class:selected={matrix[i][j].selected}
								class:toFind={pointToFind.row === i && pointToFind.column === j}
								class:toStart={pointToStart.row === i && pointToStart.column === j}
								class:pointFound={matrix[i][j].selected &&
									pointToFind.row === i &&
									pointToFind.column === j}
								transition:scale={{
									duration: isDraggingPointToFind || executing ? 0 : 200,
									delay: 0,
									start: isDraggingPointToFind && executing ? 1 : 0,
									easing: quintOut
								}}
								style={`cursor: ${
									pointToFind.row === i &&
									pointToFind.column === j &&
									!executing &&
									ALGORITHMS[selectedAlgo].section === SECTIONS.maze
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
	on:mouseover={() => {
		dragging = false;
		isDraggingPointToFind = false;
	}}
	on:mouseleave={() => {
		dragging = false;
		isDraggingPointToFind = false;
	}} />

<style>
	button {
		display: inline-block;
		padding: 0.5em 1.2em;
		margin: 0 0.3em 0.3em 0;
		border-radius: 4px;
		box-sizing: border-box;
		text-decoration: none;
		font-weight: 300;
		color: #ffffff;
		background-color: #4eb5f1;
		text-align: center;
		transition: all 0.2s;
		border: none;
		cursor: pointer;
	}
	button:hover:not([disabled]) {
		background-color: #4095c6;
	}

	button:disabled {
		background-color: #29678b;
		cursor: default;
	}

	.header {
		width: 100%;
		display: flex;
		justify-content: center;
		flex-direction: column;
		align-items: center;
		height: 250px;
		background: #181c25;
	}
	.header div:last-child {
		margin: 40px 0;
	}

	main {
		text-align: center;
		-webkit-user-select: none;
		display: grid;
		place-items: center;
		background: #293241;
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
		font-size: 0.75em;
	}
	.thInner {
		display: block;
		width: 40px;
		height: 40px;
		position: absolute;
		margin: 0;
		top: 0;
	}

	.valueDiv {
		display: block;
		width: 40px;
		height: 40px;
		position: absolute;
		margin: 0;
		top: 0;
		z-index: 2;
		display: grid;
		place-items: center;
	}
	.selected {
		background: #3d5a80;
	}
	.toFind {
		background: red;
	}
	.toStart {
		background: purple;
	}
	.pointFound {
		background: green;
	}
</style>
