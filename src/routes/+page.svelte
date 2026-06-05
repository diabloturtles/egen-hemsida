<script>
	// board config — changes when a preset is selected
	let rows = $state(9);
	let cols = $state(9);
	let mineCount = $state(10);

	// game state
	let data = $state([]);
	let isGameOver = $state(false);
	let isGameWon = $state(false);
	let timerCount = $state(0);
	let hasStarted = false;
	let minesPlaced = false;
	let interval = null;
	let startTime = 0;

	// dropdown menu
	let menuOpen = $state(false);

	const presets = [
		{ label: 'Nybörjare  (9×9,   10 minor)', rows: 9,  cols: 9,  mines: 10 },
		{ label: 'Medel      (16×16, 40 minor)', rows: 16, cols: 16, mines: 40 },
		{ label: 'Expert     (30×16, 99 minor)', rows: 16, cols: 30, mines: 99 }
	];

	function selectPreset(preset) {
		rows = preset.rows;
		cols = preset.cols;
		mineCount = preset.mines;
		menuOpen = false;
		resetGame();
	}

	// window width adapts to the board size
	let gameWidth = $derived(Math.max(cols * 32 + 22, 260));

	// ── Statistics ──
	let totalBV    = $state(0);
	let solvedBV   = $state(0);
	let clicksLeft   = $state(0);
	let clicksRight  = $state(0);
	let clicksDouble = $state(0);
	let bvComponents = []; // not reactive — just used for calculations

	let totalClicks = $derived(clicksLeft + clicksRight + clicksDouble);
	let bvPerSec    = $derived(timerCount > 0 ? solvedBV / timerCount : 0);
	let cps         = $derived(timerCount > 0 ? totalClicks / timerCount : 0);
	let stnb        = $derived(cps > 0 ? bvPerSec / cps : 0);
	let ioe         = $derived(totalClicks > 0 ? solvedBV / totalClicks : 0);

	// flags set at win-time to show PR badges
	let isNewTimeRecord = $state(false);
	let isNewBvsRecord  = $state(false);

	// jumpscare (förlust), win-scare och flag-scare
	let showJumpscare  = $state(false);
	let showWinscare   = $state(false);
	let showFlagScare  = $state(false);
	let showFlagScare2 = $state(false);
	let showFlagScare3 = $state(false);
	let flagScare3Text = $state('');
	let flagAttempts   = 0;

	// Noah video
	let showNoahVideo = $state(false);

	function boostVolume(node) {
		const ctx = new AudioContext();
		const source = ctx.createMediaElementSource(node);
		const gain = ctx.createGain();
		gain.gain.value = 3;
		source.connect(gain);
		gain.connect(ctx.destination);
		return { destroy() { ctx.close(); } };
	}

	// Noah-hjälp knapp
	let noahHelpCount = $state(0);
	let showNoahBoll  = $state(false);
	let showNoahArg2  = $state(false);
	let showNoahFinal  = $state(false);
	let noahFinalText  = $state('Du har begått ditt sista misstag');
	let bollAttackText = $state(false);
	let fallingBalls   = $state([]);
	let ballIdCounter  = 0;

	function startBollAttack() {
		bollAttackText = true;
		setTimeout(() => {
			bollAttackText = false;
			launchBallsOnMines();
		}, 1500);
	}

	function launchBallsOnMines() {
		let mines = [];
		for (let i = 0; i < data.length; i++) {
			if (data[i].mine) mines.push(i);
		}

		for (let i = 0; i < mines.length; i++) {
			setTimeout(() => {
				let ballId = ballIdCounter++;
				let xPos = 5 + Math.random() * 85;
				fallingBalls = [...fallingBalls, { id: ballId, x: xPos }];

				setTimeout(() => {
					data[mines[i]] = { ...data[mines[i]], open: true };
					fallingBalls = fallingBalls.filter(b => b.id !== ballId);

					if (i == mines.length - 1) {
						isGameOver = true;
						clearInterval(interval);
						noahFinalText = 'Den som skrattar bäst skrattar sist muhahaha';
						showNoahFinal = true;
						setTimeout(() => { showNoahFinal = false; }, 3000);
					}
				}, 1200);
			}, i * 350);
		}
	}

	function handleNoahHelp() {
		noahHelpCount = noahHelpCount + 1;

		if (noahHelpCount == 1) {
			if (!minesPlaced) { noahHelpCount = 0; return; }
			let safeCells = [];
			for (let i = 0; i < data.length; i++) {
				if (!data[i].mine && !data[i].open && data[i].count > 0) safeCells.push(i);
			}
			if (safeCells.length == 0) { noahHelpCount = 0; return; }
			let pick = safeCells[Math.floor(Math.random() * safeCells.length)];
			data[pick] = { ...data[pick], open: true };
			updateSolvedBV();
		} else if (noahHelpCount == 2) {
			showNoahBoll = true;
			setTimeout(() => { showNoahBoll = false; }, 3000);
		} else if (noahHelpCount == 3) {
			showNoahArg2 = true;
			setTimeout(() => { showNoahArg2 = false; }, 3000);
		} else {
			showNoahFinal = true;
			setTimeout(() => {
				showNoahFinal = false;
				if (minesPlaced) startBollAttack();
			}, 3000);
		}
	}

	function handleRightClick(e) {
		e.preventDefault();
		flagAttempts = flagAttempts + 1;
		if (flagAttempts >= 3) {
			triggerMonster();
		} else if (flagAttempts >= 2) {
			showFlagScare2 = true;
			setTimeout(() => { showFlagScare2 = false; }, 3000);
		} else {
			showFlagScare = true;
			setTimeout(() => { showFlagScare = false; }, 3000);
		}
	}

	function triggerMonster() {
		flagScare3Text = 'Nu har du väckt ett monster';
		showFlagScare3 = true;

		setTimeout(() => {
			showFlagScare3 = false;
			if (!minesPlaced) return;
			openSafeTilesAndBlow();
		}, 3000);
	}

	function openSafeTilesAndBlow() {
		let safeCells = [];
		let mineCells = [];
		for (let i = 0; i < data.length; i++) {
			if (!data[i].mine && !data[i].open && data[i].count > 0) safeCells.push(i);
			if (data[i].mine) mineCells.push(i);
		}

		let openedSafe = 0;
		for (let i = 0; i < data.length; i++) {
			if (!data[i].mine && data[i].open) openedSafe++;
		}

		let totalSafe    = rows * cols - mineCount;
		let maxCanOpen   = totalSafe - openedSafe - 1; // -1 så vi inte vinner
		let toOpen       = Math.min(5, Math.max(0, maxCanOpen));
		let cellsToOpen  = safeCells.slice(0, toOpen);

		if (mineCells.length == 0) return;

		if (cellsToOpen.length == 0) {
			blowMine(mineCells[0]);
			return;
		}

		for (let i = 0; i < cellsToOpen.length; i++) {
			setTimeout(() => {
				data[cellsToOpen[i]] = { ...data[cellsToOpen[i]], open: true };
				updateSolvedBV();
				if (i == cellsToOpen.length - 1) {
					setTimeout(() => blowMine(mineCells[0]), 500);
				}
			}, i * 1000);
		}
	}

	function blowMine(mineIndex) {
		if (isGameOver) return;
		data[mineIndex] = { ...data[mineIndex], open: true };
		isGameOver = true;
		clearInterval(interval);
		setTimeout(() => {
			flagScare3Text = 'Väck mig inte igen';
			showFlagScare3 = true;
			setTimeout(() => { showFlagScare3 = false; }, 3000);
		}, 300);
	}

	// per-difficulty best stats saved in localStorage
	let bestStats = $state(loadBestStats());

	function loadBestStats() {
		try {
			let saved = localStorage.getItem('minesweeper-best-stats');
			if (saved != null) return JSON.parse(saved);
		} catch (e) {}
		return {};
	}

	function getDifficultyLabel() {
		if (rows == 9  && cols == 9  && mineCount == 10) return 'Nybörjare';
		if (rows == 16 && cols == 16 && mineCount == 40) return 'Medel';
		if (rows == 16 && cols == 30 && mineCount == 99) return 'Expert';
		return rows + '×' + cols;
	}

	function getDifficultyKey() {
		if (rows == 9  && cols == 9  && mineCount == 10) return 'nybörjare';
		if (rows == 16 && cols == 16 && mineCount == 40) return 'medel';
		if (rows == 16 && cols == 30 && mineCount == 99) return 'expert';
		return 'anpassad';
	}

	// reactive — re-evaluates whenever bestStats or board dims change
	let currentBest = $derived(bestStats[getDifficultyKey()] || null);

	function saveBestStats(seconds, bvs) {
		let key = getDifficultyKey();
		let existing = bestStats[key] || { time: null, bvs: null };
		let newTime = existing.time == null || seconds < existing.time ? seconds : existing.time;
		let newBvs  = existing.bvs  == null || bvs > existing.bvs    ? bvs    : existing.bvs;
		let updated = { ...bestStats, [key]: { time: newTime, bvs: newBvs } };
		bestStats = updated;
		try {
			localStorage.setItem('minesweeper-best-stats', JSON.stringify(updated));
		} catch (e) {}
	}

	// 3BV: count minimum clicks needed to clear the board
	// each "opening" (connected group of 0-cells) = 1, each isolated numbered cell = 1
	function calculate3BV() {
		bvComponents = [];
		let coveredByOpening = new Array(rows * cols).fill(false);
		let visitedOpening   = new Array(rows * cols).fill(false);

		for (let i = 0; i < data.length; i++) {
			if (data[i].mine)     continue;
			if (data[i].count != 0) continue;
			if (visitedOpening[i])  continue;

			let component = [];
			let queue = [i];
			visitedOpening[i] = true;

			while (queue.length > 0) {
				let curr = queue.shift();
				component.push(curr);
				coveredByOpening[curr] = true;

				let r = data[curr].row;
				let c = data[curr].col;

				for (let dr = -1; dr <= 1; dr++) {
					for (let dc = -1; dc <= 1; dc++) {
						if (dr == 0 && dc == 0) continue;
						let nr = r + dr;
						let nc = c + dc;
						if (nr < 0 || nr >= rows || nc < 0 || nc >= cols) continue;
						let next = nr * cols + nc;
						if (data[next].mine) continue;
						coveredByOpening[next] = true;
						if (data[next].count != 0) continue;
						if (visitedOpening[next]) continue;
						visitedOpening[next] = true;
						queue.push(next);
					}
				}
			}

			bvComponents.push({ cells: component });
		}

		// numbered cells not adjacent to any opening each require their own click
		for (let i = 0; i < data.length; i++) {
			if (data[i].mine)       continue;
			if (data[i].count == 0) continue;
			if (coveredByOpening[i]) continue;
			bvComponents.push({ cells: [i] });
		}

		totalBV = bvComponents.length;
	}

	function updateSolvedBV() {
		let solved = 0;
		for (let i = 0; i < bvComponents.length; i++) {
			let comp = bvComponents[i];
			for (let j = 0; j < comp.cells.length; j++) {
				if (data[comp.cells[j]].open == true) {
					solved++;
					break;
				}
			}
		}
		solvedBV = solved;
	}

	// start the game right away
	startNewGame();

	function startNewGame() {
		data = [];

		// create all cells
		for (let i = 0; i < rows * cols; i++) {
			data.push({
				index: i,
				row: Math.floor(i / cols),
				col: i % cols,
				mine: false,
				open: false,
				count: 0
			});
		}

		// mines are placed on the first click so the player can never die immediately
		minesPlaced = false;
	}

	function placeMines(safeIndex) {
		let placed = 0;
		while (placed < mineCount) {
			let pos = Math.floor(Math.random() * (rows * cols));
			if (pos == safeIndex) continue;
			if (data[pos].mine == false) {
				data[pos].mine = true;
				placed = placed + 1;
			}
		}

		for (let i = 0; i < data.length; i++) {
			if (data[i].mine) continue;

			let total = 0;
			let r = data[i].row;
			let c = data[i].col;

			for (let dr = -1; dr <= 1; dr++) {
				for (let dc = -1; dc <= 1; dc++) {
					if (dr == 0 && dc == 0) continue;
					let nr = r + dr;
					let nc = c + dc;
					if (nr < 0 || nr >= rows || nc < 0 || nc >= cols) continue;
					if (data[nr * cols + nc].mine) total++;
				}
			}

			data[i].count = total;
		}

		minesPlaced = true;
		calculate3BV();
	}

	function handleClick(index) {
		if (isGameOver || isGameWon) return;
		if (data[index].open) return;

		clicksLeft = clicksLeft + 1;

		// start the timer and place mines on the very first click
		if (hasStarted == false) {
			hasStarted = true;
			placeMines(index);
			startTime = Date.now();
			interval = setInterval(function() {
				timerCount = Date.now() - startTime;
			}, 10);
		}

		// if they clicked a mine it's game over
		if (data[index].mine == true) {
			data[index].open = true;
			isGameOver = true;
			clearInterval(interval);
			showJumpscare = true;
			setTimeout(() => { showJumpscare = false; }, 3000);
			return;
		}

		// use a queue to open up connected empty cells (BFS)
		let queue = [];
		let seen = [];
		queue.push(index);
		seen.push(index);

		while (queue.length > 0) {
			let curr = queue.shift();
			data[curr].open = true;

			// only spread further if this cell has no mines nearby
			if (data[curr].count == 0) {
				let r = data[curr].row;
				let c = data[curr].col;

				for (let dr = -1; dr <= 1; dr++) {
					for (let dc = -1; dc <= 1; dc++) {
						if (dr == 0 && dc == 0) continue;
						let nr = r + dr;
						let nc = c + dc;
						if (nr < 0 || nr >= rows || nc < 0 || nc >= cols) continue;
						let next = nr * cols + nc;
						if (seen.includes(next)) continue;
						if (data[next].mine) continue;
						seen.push(next);
						queue.push(next);
					}
				}
			}
		}

		updateSolvedBV();

		// count how many safe cells have been opened
		let openCount = 0;
		for (let i = 0; i < data.length; i++) {
			if (data[i].open == true && data[i].mine == false) {
				openCount = openCount + 1;
			}
		}
		if (openCount == rows * cols - mineCount) {
			isGameWon = true;
			clearInterval(interval);

			// check for new records before saving (so we know what to highlight)
			let key = getDifficultyKey();
			let existing = bestStats[key] || null;
			isNewTimeRecord = existing == null || timerCount < existing.time;
			isNewBvsRecord  = existing == null || bvPerSec > (existing.bvs || 0);

			saveToLeaderboard(timerCount);
			saveBestStats(timerCount, bvPerSec);
			showWinscare = true;
			setTimeout(() => { showWinscare = false; }, 3000);
		}
	}

	function resetGame() {
		isGameOver = false;
		isGameWon  = false;
		timerCount = 0;
		hasStarted = false;
		clearInterval(interval);
		interval = null;
		clicksLeft   = 0;
		clicksRight  = 0;
		clicksDouble = 0;
		solvedBV = 0;
		totalBV  = 0;
		bvComponents = [];
		isNewTimeRecord = false;
		isNewBvsRecord  = false;
		showJumpscare  = false;
		showWinscare   = false;
		showFlagScare  = false;
		showFlagScare2 = false;
		showFlagScare3 = false;
		flagScare3Text = '';
		flagAttempts   = 0;
		noahHelpCount  = 0;
		showNoahBoll   = false;
		showNoahArg2   = false;
		showNoahFinal  = false;
		noahFinalText  = 'Du har begått ditt sista misstag';
		bollAttackText = false;
		fallingBalls   = [];
		minesPlaced = false;
		startNewGame();
	}

	// ── Leaderboard ──

	let leaderboard = $state(loadLeaderboard());

	function loadLeaderboard() {
		try {
			let saved = localStorage.getItem('minesweeper-leaderboard');
			if (saved != null) {
				return JSON.parse(saved);
			}
		} catch (e) {}
		return [];
	}

	function saveToLeaderboard(seconds) {
		if (leaderboard.length >= 10 && seconds > leaderboard[leaderboard.length - 1].time) {
			return;
		}

		let newEntry = {
			time: seconds,
			date: new Date().toLocaleDateString(),
			board: getDifficultyLabel()
		};

		let updated = [...leaderboard, newEntry];
		updated.sort(function(a, b) { return a.time - b.time; });
		leaderboard = updated.slice(0, 10);

		try {
			localStorage.setItem('minesweeper-leaderboard', JSON.stringify(leaderboard));
		} catch (e) {}
	}

	function clearLeaderboard() {
		leaderboard = [];
		try {
			localStorage.removeItem('minesweeper-leaderboard');
		} catch (e) {}
	}

	function toRoman(num) {
		const roman = ['', 'I', 'II', 'III', 'IV', 'V', 'VI', 'VII', 'VIII'];
		return roman[num];
	}

	// returns the right color for each number like in real minesweeper
	function getColor(num) {
		if (num == 1) return 'blue';
		if (num == 2) return 'green';
		if (num == 3) return 'red';
		if (num == 4) return 'darkblue';
		if (num == 5) return 'darkred';
		if (num == 6) return 'teal';
		if (num == 7) return 'black';
		if (num == 8) return 'gray';
		return '';
	}
</script>

<!-- close the menu when clicking anywhere outside it -->
<svelte:window onclick={() => (menuOpen = false)} onkeydown={(e) => { if (e.code === 'Space') { e.preventDefault(); resetGame(); } }} />

{#if showJumpscare}
	<div class="jumpscare" onclick={() => (showJumpscare = false)} role="presentation" onkeydown={() => {}}>
		<img src="/noah.jpg" alt="Noah är arg" class="jumpscare-img" />
		<p class="jumpscare-text">Noah är besviken</p>
	</div>
{/if}

{#if showFlagScare}
	<div class="jumpscare flagscare" onclick={() => (showFlagScare = false)} role="presentation" onkeydown={() => {}}>
		<img src="/flagga.jpg" alt="Inga flaggor" class="jumpscare-img" />
		<p class="jumpscare-text flagscare-text">Noah gillar inte flaggor testa inte honom</p>
	</div>
{/if}

{#if showNoahVideo}
	<div class="video-overlay" onclick={() => (showNoahVideo = false)} role="presentation" onkeydown={() => {}}>
		<div class="video-box" onclick={(e) => e.stopPropagation()} role="presentation" onkeydown={() => {}}>
			<div class="video-titlebar">
				<span>🎬 Noah</span>
				<button class="video-close-btn" onclick={() => (showNoahVideo = false)}>✕</button>
			</div>
			<!-- svelte-ignore a11y_media_has_caption -->
			<video class="noah-video" src="/video.mp4" controls autoplay use:boostVolume></video>
		</div>
	</div>
{/if}

{#each fallingBalls as ball (ball.id)}
	<img
		src="/boll.jpg"
		alt="boll"
		class="falling-ball"
		style="left: {ball.x}%; animation-duration: 1.2s;"
	/>
{/each}

{#if showNoahFinal}
	<div class="jumpscare finalscare" role="presentation" onkeydown={() => {}} onclick={() => {}}>
		<div class="explosion" style="top:4%;  left:6%;  animation-delay:0.0s; font-size:3.5rem">💥</div>
		<div class="explosion" style="top:8%;  left:78%; animation-delay:0.3s; font-size:3rem">💥</div>
		<div class="explosion" style="top:70%; left:5%;  animation-delay:0.5s; font-size:4rem">💥</div>
		<div class="explosion" style="top:75%; left:80%; animation-delay:0.2s; font-size:3rem">💥</div>
		<div class="explosion" style="top:45%; left:90%; animation-delay:0.7s; font-size:2.5rem">💥</div>
		<div class="explosion" style="top:85%; left:45%; animation-delay:0.1s; font-size:3rem">💥</div>
		<div class="lightning" style="left:12%; animation-delay:0.0s"></div>
		<div class="lightning" style="left:45%; animation-delay:0.35s"></div>
		<div class="lightning" style="left:75%; animation-delay:0.15s"></div>
		<div class="lightning" style="left:28%; animation-delay:0.55s"></div>
		<div class="lightning" style="left:62%; animation-delay:0.7s"></div>
		<img src="/noah_final.jpg" alt="Noah final" class="jumpscare-img final-img" />
		<p class="jumpscare-text final-text">{noahFinalText}</p>
	</div>
{/if}

{#if showNoahArg2}
	<div class="jumpscare" role="presentation" onkeydown={() => {}} onclick={() => {}}>
		<img src="/noah_arg2.jpg" alt="Noah arg" class="jumpscare-img" />
		<p class="jumpscare-text">Oj nu blev någon arg</p>
	</div>
{/if}

{#if showNoahBoll}
	<div class="jumpscare bollscare" role="presentation" onkeydown={() => {}} onclick={() => {}}>
		<img src="/noah_boll.jpg" alt="Noah i bollhavet" class="jumpscare-img" />
		<p class="jumpscare-text boll-text">Noah är just nu i bollhavet</p>
	</div>
{/if}

{#if showFlagScare3}
	<div class="jumpscare monsterscare" role="presentation" onkeydown={() => {}} onclick={() => {}}>
		<img src="/noah_monster.jpg" alt="monster" class="jumpscare-img monster-img" />
		<p class="jumpscare-text monster-text">{flagScare3Text}</p>
	</div>
{/if}

{#if showFlagScare2}
	<div class="jumpscare flagscare2" onclick={() => (showFlagScare2 = false)} role="presentation" onkeydown={() => {}}>
		<div class="explosion" style="top:5%;  left:8%;  animation-delay:0.0s">💥</div>
		<div class="explosion" style="top:10%; left:75%; animation-delay:0.2s">💥</div>
		<div class="explosion" style="top:30%; left:5%;  animation-delay:0.4s">💥</div>
		<div class="explosion" style="top:20%; left:55%; animation-delay:0.1s">💥</div>
		<div class="explosion" style="top:60%; left:80%; animation-delay:0.3s">💥</div>
		<div class="explosion" style="top:70%; left:15%; animation-delay:0.5s">💥</div>
		<div class="explosion" style="top:80%; left:50%; animation-delay:0.2s">💥</div>
		<div class="explosion" style="top:50%; left:30%; animation-delay:0.6s">💥</div>
		<div class="explosion" style="top:85%; left:85%; animation-delay:0.1s">💥</div>
		<div class="explosion" style="top:40%; left:90%; animation-delay:0.4s">💥</div>
		<img src="/noah_varnade.jpg" alt="Noah varnade dig" class="jumpscare-img" />
		<p class="jumpscare-text flagscare2-text">Noah varnade dig</p>
	</div>
{/if}

{#if showWinscare}
	<div class="jumpscare winscare" onclick={() => (showWinscare = false)} role="presentation" onkeydown={() => {}}>
		<img src="/noah_stolt.jpg" alt="Noah är stolt" class="jumpscare-img" />
		<p class="jumpscare-text winscare-text">Noah är stolt över dig</p>
	</div>
{/if}

<div class="layout-row">

<div class="help-col">
	<button class="noah-help-btn" class:used={noahHelpCount >= 1} onclick={handleNoahHelp}>
		 Be guden Noah om hjälp 🙏
	</button>
	<button class="noah-help-btn" onclick={() => (showNoahVideo = true)}>
		 Se en video om Noah 🎬
	</button>
</div>

<div class="game-col">
<h1 style="width: {gameWidth}px">Noahsweeper</h1>

<main style="width: {gameWidth}px">
	<!-- top bar -->
	<div class="topbar">

		<!-- left section: difficulty menu + mine counter -->
		<div class="topbar-left">
			<!-- clicking inside the wrapper stops the event reaching svelte:window -->
			<div class="menu-wrapper" role="presentation" onclick={(e) => e.stopPropagation()} onkeydown={(e) => e.stopPropagation()}>
				<button class="menu-btn" onclick={() => (menuOpen = !menuOpen)}>▼</button>
				{#if menuOpen}
					<div class="menu-dropdown">
						{#each presets as preset}
							<button class="menu-item" onclick={() => selectPreset(preset)}>
								{preset.label}
							</button>
						{/each}
					</div>
				{/if}
			</div>
			<span class="counter">💣 {mineCount}</span>
		</div>

		<!-- center: reset / face button -->
		<button class="face-btn" onclick={resetGame}>
			{#if isGameWon}
				😎
			{:else if isGameOver}
				😵
			{:else}
				🙂
			{/if}
		</button>

		<!-- right: timer -->
		<span class="counter">⏱ {(timerCount / 1000).toFixed(2)}s</span>
	</div>

	<!-- show a message if the game ended -->
	{#if isGameWon}
		<p class="message win">Hej Noah! Cleared in {(timerCount / 1000).toFixed(2)} seconds!</p>
	{/if}
	{#if isGameOver}
		<p class="message lose">Game over! 💥 You hit a mine!</p>
	{/if}

	<!-- grid of cells — columns and rows set dynamically via inline style -->
	<div
		class="grid"
		style="grid-template-columns: repeat({cols}, 32px); grid-template-rows: repeat({rows}, 32px);"
	>
		{#each data as cell, i}
			<button
				class="cell"
				class:revealed={cell.open}
				class:is-mine={cell.open && cell.mine}
				style="color: {cell.open ? getColor(cell.count) : ''};"
				onclick={() => handleClick(i)}
				oncontextmenu={handleRightClick}
				disabled={isGameOver || isGameWon}
			>
				{#if cell.open}
					{#if cell.mine}
						💣
					{:else if cell.count > 0}
						{toRoman(cell.count)}
					{/if}
				{/if}
			</button>
		{/each}
	</div>
</main>

<div class="lb-wrapper">
<div class="lb-titlebar">🏆 Best Times</div>
<section class="leaderboard">
	{#if leaderboard.length == 0}
		<p class="lb-empty">No times yet. Win a game to get on the board!</p>
	{:else}
		<table class="lb-table">
			<thead>
				<tr>
					<th>#</th>
					<th>Time</th>
					<th>Board</th>
					<th>Date</th>
				</tr>
			</thead>
			<tbody>
				{#each leaderboard as entry, i}
					<tr>
						<td>{i + 1}</td>
						<td>{(entry.time / 1000).toFixed(2)}s</td>
						<td>{entry.board || '?'}</td>
						<td>{entry.date}</td>
					</tr>
				{/each}
			</tbody>
		</table>
	{/if}
	<button class="clear-btn" onclick={clearLeaderboard}>Clear Leaderboard</button>
</section>
</div>

</div>

<div class="stats-col">
<!-- ── Statistics panel ── -->
<div class="stats-titlebar">📊 Statistik</div>
<section class="stats-panel">

	<div class="stats-grid">
		<div class="stat-item">
			<span class="stat-label">3BV</span>
			<span class="stat-value">{totalBV}</span>
		</div>
		<div class="stat-item">
			<span class="stat-label">3BV/s</span>
			<span class="stat-value">{bvPerSec.toFixed(3)}</span>
		</div>
		<div class="stat-item">
			<span class="stat-label">Löst 3BV</span>
			<span class="stat-value">{solvedBV}</span>
		</div>
		<div class="stat-item">
			<span class="stat-label">Klick L</span>
			<span class="stat-value">{clicksLeft}</span>
		</div>
		<div class="stat-item">
			<span class="stat-label">CPS</span>
			<span class="stat-value">{cps.toFixed(3)}</span>
		</div>
		<div class="stat-item">
			<span class="stat-label">IOE</span>
			<span class="stat-value">{ioe.toFixed(3)}</span>
		</div>
		<div class="stat-item">
			<span class="stat-label">STNB</span>
			<span class="stat-value">{stnb.toFixed(3)}</span>
		</div>
		<div class="stat-item">
			<span class="stat-label">Klick R</span>
			<span class="stat-value">{clicksRight}</span>
		</div>
		<div class="stat-item">
			<span class="stat-label">Klick D</span>
			<span class="stat-value">{clicksDouble}</span>
		</div>
		<div class="stat-item">
			<span class="stat-label">Bästa tid</span>
			<span class="stat-value">{currentBest ? (currentBest.time / 1000).toFixed(2) + 's' : '—'}</span>
		</div>
		<div class="stat-item">
			<span class="stat-label">Bästa 3BV/s</span>
			<span class="stat-value">{currentBest ? currentBest.bvs.toFixed(3) : '—'}</span>
		</div>
		<div class="stat-item stat-diff">
			<span class="stat-label">Nivå</span>
			<span class="stat-value">{getDifficultyKey()}</span>
		</div>
	</div>

	{#if isGameWon}
		<div class="win-summary">
			<div class="ws-title">✓ Omgångssummering</div>
			<div class="ws-row">
				<span class="ws-label">Tid</span>
				<span class="ws-val">{(timerCount / 1000).toFixed(2)}s {#if isNewTimeRecord}<span class="pr-badge">PR!</span>{/if}</span>
			</div>
			<div class="ws-row">
				<span class="ws-label">3BV/s</span>
				<span class="ws-val">{bvPerSec.toFixed(3)} {#if isNewBvsRecord}<span class="pr-badge">PR!</span>{/if}</span>
			</div>
			<div class="ws-row">
				<span class="ws-label">Löst 3BV</span>
				<span class="ws-val">{solvedBV} / {totalBV}</span>
			</div>
			<div class="ws-row">
				<span class="ws-label">Klick (L)</span>
				<span class="ws-val">{totalClicks}</span>
			</div>
			<div class="ws-row">
				<span class="ws-label">CPS</span>
				<span class="ws-val">{cps.toFixed(3)}</span>
			</div>
			<div class="ws-row">
				<span class="ws-label">IOE</span>
				<span class="ws-val">{ioe.toFixed(3)}</span>
			</div>
			<div class="ws-row">
				<span class="ws-label">STNB</span>
				<span class="ws-val">{stnb.toFixed(3)}</span>
			</div>
			{#if currentBest}
				<div class="ws-divider"></div>
				<div class="ws-row ws-pr-row">
					<span class="ws-label">PR tid</span>
					<span class="ws-val">{(currentBest.time / 1000).toFixed(2)}s</span>
				</div>
				<div class="ws-row ws-pr-row">
					<span class="ws-label">PR 3BV/s</span>
					<span class="ws-val">{currentBest.bvs.toFixed(3)}</span>
				</div>
			{/if}
		</div>
	{/if}

</section>
</div>
</div>

<style>
	/* ── Page ── */
	:global(body) {
		margin: 0;
		padding: 24px 16px;
		min-height: 100vh;
		background-color: #008080;
		display: flex;
		flex-direction: column;
		align-items: center;
		font-family: "MS Sans Serif", Arial, sans-serif;
		box-sizing: border-box;
	}

	/* ── Jumpscare overlay ── */
	:global(.jumpscare) {
		position: fixed;
		top: 0;
		left: 0;
		width: 100%;
		height: 100%;
		background-color: #000000;
		display: flex;
		flex-direction: column;
		align-items: center;
		justify-content: center;
		z-index: 9999;
		cursor: pointer;
		animation: scare 0.08s ease-out;
	}

	:global(.jumpscare-img) {
		max-width: 90%;
		max-height: 75vh;
		object-fit: contain;
	}

	:global(.jumpscare-text) {
		color: #ff0000;
		font-family: Impact, Arial, sans-serif;
		font-size: 4rem;
		font-weight: bold;
		margin: 24px 0 0;
		text-shadow: 4px 4px 0px #000000;
		letter-spacing: 0.05em;
		text-transform: uppercase;
	}

	:global(.monsterscare) {
		background-color: #000010;
		cursor: default;
	}

	:global(.monster-img) {
		animation: monsterPulse 1.5s ease-in-out infinite;
	}

	:global(.monster-text) {
		color: #00cfff;
		text-shadow: 0 0 30px #00cfff, 0 0 60px #0055ff, 3px 3px 0px #000000;
		animation: textshake 0.08s infinite;
	}

	@keyframes -global-monsterPulse {
		0%   { filter: brightness(1)   drop-shadow(0 0 0px #00cfff); }
		50%  { filter: brightness(1.2) drop-shadow(0 0 30px #00cfff); }
		100% { filter: brightness(1)   drop-shadow(0 0 0px #00cfff); }
	}

	:global(.boll-attack-banner) {
		position: fixed;
		top: 0; left: 0;
		width: 100%; height: 100%;
		display: flex;
		align-items: center;
		justify-content: center;
		font-family: Impact, Arial, sans-serif;
		font-size: 5rem;
		font-weight: bold;
		color: #00aaff;
		text-shadow: 0 0 30px #00aaff, 0 0 60px #ffffff, 4px 4px 0px #000044;
		background-color: rgba(0, 0, 30, 0.85);
		z-index: 9998;
		animation: textshake 0.08s infinite;
	}

	:global(.falling-ball) {
		position: fixed;
		top: -100px;
		width: 70px;
		height: 70px;
		object-fit: contain;
		z-index: 9997;
		animation: fallDown 1.2s ease-in forwards;
		pointer-events: none;
	}

	@keyframes -global-fallDown {
		0%   { top: -100px; transform: rotate(0deg);   }
		100% { top: 110vh;  transform: rotate(720deg); }
	}

	:global(.lightning) {
		position: absolute;
		top: 0;
		width: 3px;
		height: 100%;
		background: linear-gradient(to bottom, transparent 0%, #ffffff 20%, #aaddff 40%, transparent 60%);
		animation: lightningFlash 0.5s ease-in-out infinite;
		pointer-events: none;
		filter: blur(1px);
	}

	@keyframes -global-lightningFlash {
		0%   { opacity: 0; transform: scaleX(1); }
		10%  { opacity: 1; transform: scaleX(2); }
		20%  { opacity: 0.3; }
		30%  { opacity: 1; transform: scaleX(1.5); }
		100% { opacity: 0; }
	}

	:global(.finalscare) {
		background-color: #080008;
	}

	:global(.final-img) {
		animation: monsterPulse 0.8s ease-in-out infinite;
		filter: drop-shadow(0 0 20px #ff0000);
	}

	:global(.final-text) {
		color: #ff2200;
		text-shadow: 0 0 20px #ff0000, 0 0 50px #ff4400, 3px 3px 0px #000000;
		animation: textshake 0.06s infinite;
	}

	:global(.flagscare2) {
		background-color: #0a0000;
	}

	:global(.flagscare2-text) {
		color: #ff0000;
		text-shadow: 0 0 20px #ff0000, 3px 3px 0px #000000;
		animation: textshake 0.1s infinite;
	}

	:global(.explosion) {
		position: absolute;
		font-size: 4rem;
		animation: explode 0.6s ease-out infinite;
		pointer-events: none;
	}

	@keyframes -global-explode {
		0%   { transform: scale(0.2); opacity: 1; }
		60%  { transform: scale(1.4); opacity: 1; }
		100% { transform: scale(1.0); opacity: 0.7; }
	}

	@keyframes -global-textshake {
		0%   { transform: translateX(-3px); }
		50%  { transform: translateX(3px); }
		100% { transform: translateX(-3px); }
	}

	:global(.flagscare) {
		background-color: #1a1a1a;
	}

	:global(.flagscare-text) {
		color: #ffff00;
		text-shadow: 3px 3px 0px #ff6600;
	}

	:global(.winscare) {
		background-color: #ff69b4;
	}

	:global(.winscare-text) {
		color: #ffffff;
		text-shadow: 3px 3px 0px #c2185b;
	}

	@keyframes -global-scare {
		0%   { transform: scale(1.15); opacity: 0; }
		100% { transform: scale(1);    opacity: 1; }
	}

	/* ── Noah video overlay ── */
	:global(.video-overlay) {
		position: fixed;
		top: 0; left: 0;
		width: 100%; height: 100%;
		background-color: rgba(0, 0, 0, 0.75);
		display: flex;
		align-items: center;
		justify-content: center;
		z-index: 9999;
	}

	:global(.video-box) {
		display: flex;
		flex-direction: column;
		background-color: #c0c0c0;
		border-top: 2px solid #ffffff;
		border-left: 2px solid #ffffff;
		border-right: 2px solid #808080;
		border-bottom: 2px solid #808080;
		max-width: 90vw;
	}

	:global(.video-titlebar) {
		background-color: #000080;
		color: #ffffff;
		font-family: "MS Sans Serif", Arial, sans-serif;
		font-size: 0.8rem;
		font-weight: bold;
		padding: 3px 6px;
		display: flex;
		justify-content: space-between;
		align-items: center;
	}

	:global(.video-close-btn) {
		background: #c0c0c0;
		border-top: 1px solid #ffffff;
		border-left: 1px solid #ffffff;
		border-right: 1px solid #808080;
		border-bottom: 1px solid #808080;
		color: #000000;
		font-size: 0.7rem;
		width: 16px;
		height: 14px;
		cursor: pointer;
		padding: 0;
		line-height: 1;
	}

	:global(.noah-video) {
		max-width: 80vw;
		max-height: 75vh;
		display: block;
	}

	/* ── Noah hjälp-knapp ── */
	.help-col {
		display: flex;
		flex-direction: column;
		align-items: center;
		justify-content: flex-start;
		padding-top: 28px;
		width: 120px;
		flex-shrink: 0;
	}

	.noah-help-btn {
		font-family: "MS Sans Serif", Arial, sans-serif;
		font-size: 0.72rem;
		font-weight: bold;
		background-color: #c0c0c0;
		border-top: 2px solid #ffffff;
		border-left: 2px solid #ffffff;
		border-right: 2px solid #808080;
		border-bottom: 2px solid #808080;
		padding: 6px 8px;
		cursor: pointer;
		text-align: center;
		line-height: 1.4;
		width: 100%;
	}

	.noah-help-btn:active {
		border-top: 2px solid #808080;
		border-left: 2px solid #808080;
		border-right: 2px solid #ffffff;
		border-bottom: 2px solid #ffffff;
	}

	.noah-help-btn.used {
		color: #808080;
		background-color: #b0b0b0;
	}

	:global(.bollscare) {
		background-color: #87ceeb;
	}

	:global(.boll-text) {
		color: #ffffff;
		text-shadow: 2px 2px 0px #1a6fa0;
	}

	/* ── Side-by-side layout: board + stats ── */
	.layout-row {
		display: flex;
		flex-direction: row;
		align-items: flex-start;
		gap: 8px;
	}

	.game-col {
		display: flex;
		flex-direction: column;
	}

	.stats-col {
		display: flex;
		flex-direction: column;
		width: 190px;
		flex-shrink: 0;
	}

	/* ── Window title bar ── */
	h1 {
		background-color: #000080;
		color: #ffffff;
		font-family: "MS Sans Serif", Arial, sans-serif;
		font-size: 0.8rem;
		font-weight: bold;
		text-align: left;
		margin: 0;
		padding: 3px 6px;
		letter-spacing: 0.02em;
		border-top: 2px solid #ffffff;
		border-left: 2px solid #ffffff;
		border-right: 2px solid #808080;
		border-bottom: none;
		box-sizing: border-box;
		user-select: none;
	}

	/* ── Window client area ── */
	main {
		display: flex;
		flex-direction: column;
		align-items: center;
		gap: 6px;
		padding: 6px;
		background-color: #c0c0c0;
		box-sizing: border-box;
		border-top: none;
		border-left: 2px solid #ffffff;
		border-right: 2px solid #808080;
		border-bottom: 2px solid #808080;
	}

	/* ── HUD / status panel ── */
	.topbar {
		display: flex;
		justify-content: space-between;
		align-items: center;
		width: 100%;
		background-color: #c0c0c0;
		border-top: 2px solid #808080;
		border-left: 2px solid #808080;
		border-right: 2px solid #ffffff;
		border-bottom: 2px solid #ffffff;
		padding: 4px 8px;
		box-sizing: border-box;
	}

	/* groups the menu button and mine counter on the left */
	.topbar-left {
		display: flex;
		align-items: center;
		gap: 4px;
		position: relative;
	}

	/* ── Difficulty menu button ── */
	.menu-btn {
		font-family: "MS Sans Serif", Arial, sans-serif;
		font-size: 0.65rem;
		width: 20px;
		height: 20px;
		background-color: #c0c0c0;
		border-top: 2px solid #ffffff;
		border-left: 2px solid #ffffff;
		border-right: 2px solid #808080;
		border-bottom: 2px solid #808080;
		cursor: pointer;
		padding: 0;
		line-height: 1;
	}

	.menu-btn:active {
		border-top: 2px solid #808080;
		border-left: 2px solid #808080;
		border-right: 2px solid #ffffff;
		border-bottom: 2px solid #ffffff;
	}

	/* ── Dropdown panel ── */
	.menu-wrapper {
		position: relative;
	}

	.menu-dropdown {
		position: absolute;
		top: calc(100% + 2px);
		left: 0;
		background-color: #c0c0c0;
		border-top: 2px solid #ffffff;
		border-left: 2px solid #ffffff;
		border-right: 2px solid #808080;
		border-bottom: 2px solid #808080;
		z-index: 100;
		display: flex;
		flex-direction: column;
		min-width: 220px;
	}

	.menu-item {
		font-family: "Courier New", monospace;
		font-size: 0.75rem;
		background-color: #c0c0c0;
		border: none;
		padding: 4px 10px;
		text-align: left;
		cursor: pointer;
		white-space: nowrap;
	}

	.menu-item:hover {
		background-color: #000080;
		color: #ffffff;
	}

	/* ── LCD counters (mine count + timer) ── */
	.counter {
		background-color: #000000;
		color: #ff0000;
		font-family: "Courier New", "Lucida Console", monospace;
		font-size: 1.3rem;
		font-weight: bold;
		letter-spacing: 0.05em;
		min-width: 56px;
		padding: 2px 4px;
		text-align: right;
		border-top: 1px solid #808080;
		border-left: 1px solid #808080;
		border-right: 1px solid #ffffff;
		border-bottom: 1px solid #ffffff;
	}

	/* ── Smiley reset button ── */
	.face-btn {
		font-size: 1.1rem;
		width: 28px;
		height: 28px;
		background-color: #c0c0c0;
		border-top: 2px solid #ffffff;
		border-left: 2px solid #ffffff;
		border-right: 2px solid #808080;
		border-bottom: 2px solid #808080;
		cursor: pointer;
		padding: 0;
		line-height: 1;
	}

	.face-btn:active {
		border-top: 2px solid #808080;
		border-left: 2px solid #808080;
		border-right: 2px solid #ffffff;
		border-bottom: 2px solid #ffffff;
	}

	/* ── Win / lose message ── */
	.message {
		font-family: "MS Sans Serif", Arial, sans-serif;
		font-size: 0.8rem;
		font-weight: bold;
		padding: 3px 8px;
		margin: 0;
		width: 100%;
		box-sizing: border-box;
		text-align: center;
		border-top: 1px solid #808080;
		border-left: 1px solid #808080;
		border-right: 1px solid #ffffff;
		border-bottom: 1px solid #ffffff;
	}

	.win  { color: #000080; background-color: #c0c0c0; }
	.lose { color: #800000; background-color: #c0c0c0; }

	/* ── Mine grid ── */
	.grid {
		display: grid;
		gap: 0;
		border-top: 3px solid #808080;
		border-left: 3px solid #808080;
		border-right: 3px solid #ffffff;
		border-bottom: 3px solid #ffffff;
		background-color: #c0c0c0;
	}

	/* ── Individual cells ── */
	.cell {
		width: 32px;
		height: 32px;
		background-color: #c0c0c0;
		border-top: 2px solid #ffffff;
		border-left: 2px solid #ffffff;
		border-right: 2px solid #808080;
		border-bottom: 2px solid #808080;
		cursor: pointer;
		font-size: 1rem;
		font-weight: bold;
		font-family: "Times New Roman", serif;
		padding: 0;
		line-height: 1;
	}

	.cell:active:not(.revealed) {
		border-top: 2px solid #808080;
		border-left: 2px solid #808080;
		border-right: 2px solid #ffffff;
		border-bottom: 2px solid #ffffff;
		background-color: #c0c0c0;
	}

	.cell.revealed {
		background-color: #c0c0c0;
		border: 1px solid #808080;
		cursor: default;
	}

	.cell.is-mine {
		background-color: #ff0000;
		border: 1px solid #808080;
	}

	.cell:disabled {
		cursor: default;
	}

	/* ── Statistics window ── */
	.stats-titlebar {
		background-color: #000080;
		color: #ffffff;
		font-family: "MS Sans Serif", Arial, sans-serif;
		font-size: 0.8rem;
		font-weight: bold;
		text-align: left;
		margin: 0;
		padding: 3px 6px;
		box-sizing: border-box;
		border-top: 2px solid #ffffff;
		border-left: 2px solid #ffffff;
		border-right: 2px solid #808080;
		border-bottom: none;
		user-select: none;
	}

	.stats-panel {
		background-color: #c0c0c0;
		padding: 6px;
		box-sizing: border-box;
		border-top: none;
		border-left: 2px solid #ffffff;
		border-right: 2px solid #808080;
		border-bottom: 2px solid #808080;
		display: flex;
		flex-direction: column;
		gap: 6px;
	}

	/* grid of stat items — 2 columns, stretches full width */
	.stats-grid {
		display: grid;
		grid-template-columns: repeat(2, 1fr);
		gap: 2px 4px;
		border-top: 2px solid #808080;
		border-left: 2px solid #808080;
		border-right: 2px solid #ffffff;
		border-bottom: 2px solid #ffffff;
		padding: 4px;
		background-color: #c0c0c0;
	}

	.stat-item {
		display: flex;
		justify-content: space-between;
		align-items: center;
		padding: 1px 4px;
		font-family: "MS Sans Serif", Arial, sans-serif;
		font-size: 0.75rem;
	}

	.stat-item:nth-child(even) {
		background-color: #d4d4d4;
	}

	/* the "Nivå" item spans both columns */
	.stat-diff {
		grid-column: 1 / -1;
		background-color: #c0c0c0;
	}

	.stat-label {
		color: #444444;
		font-weight: normal;
	}

	.stat-value {
		font-family: "Courier New", monospace;
		font-size: 0.8rem;
		font-weight: bold;
		color: #000000;
		text-align: right;
	}

	/* ── Win summary box ── */
	.win-summary {
		border-top: 2px solid #808080;
		border-left: 2px solid #808080;
		border-right: 2px solid #ffffff;
		border-bottom: 2px solid #ffffff;
		padding: 4px 6px;
		background-color: #c0c0c0;
		display: flex;
		flex-direction: column;
		gap: 2px;
	}

	.ws-title {
		font-family: "MS Sans Serif", Arial, sans-serif;
		font-size: 0.8rem;
		font-weight: bold;
		color: #000080;
		margin-bottom: 3px;
	}

	.ws-row {
		display: flex;
		justify-content: space-between;
		align-items: center;
		font-family: "MS Sans Serif", Arial, sans-serif;
		font-size: 0.75rem;
	}

	.ws-label {
		color: #444444;
	}

	.ws-val {
		font-family: "Courier New", monospace;
		font-size: 0.8rem;
		font-weight: bold;
		color: #000000;
		display: flex;
		align-items: center;
		gap: 4px;
	}

	.ws-pr-row .ws-label {
		color: #000080;
		font-weight: bold;
	}

	.ws-divider {
		height: 1px;
		background-color: #808080;
		margin: 3px 0;
	}

	/* PR badge shown next to a new personal record */
	.pr-badge {
		background-color: #000080;
		color: #ffffff;
		font-family: "MS Sans Serif", Arial, sans-serif;
		font-size: 0.65rem;
		font-weight: bold;
		padding: 1px 4px;
		letter-spacing: 0.05em;
	}

	/* ── Leaderboard wrapper (centers it under the game) ── */
	.lb-wrapper {
		display: flex;
		flex-direction: column;
		margin-top: 8px;
	}

	/* ── Leaderboard window ── */
	.lb-titlebar {
		background-color: #000080;
		color: #ffffff;
		font-family: "MS Sans Serif", Arial, sans-serif;
		font-size: 0.8rem;
		font-weight: bold;
		text-align: left;
		margin: 0;
		padding: 3px 6px;
		box-sizing: border-box;
		border-top: 2px solid #ffffff;
		border-left: 2px solid #ffffff;
		border-right: 2px solid #808080;
		border-bottom: none;
		user-select: none;
	}

	.leaderboard {
		background-color: #c0c0c0;
		padding: 6px;
		box-sizing: border-box;
		border-top: none;
		border-left: 2px solid #ffffff;
		border-right: 2px solid #808080;
		border-bottom: 2px solid #808080;
		display: flex;
		flex-direction: column;
		gap: 6px;
	}

	.lb-empty {
		font-family: "MS Sans Serif", Arial, sans-serif;
		font-size: 0.8rem;
		text-align: center;
		margin: 4px 0;
		color: #000000;
	}

	.lb-table {
		width: 100%;
		border-collapse: collapse;
		font-family: "MS Sans Serif", Arial, sans-serif;
		font-size: 0.8rem;
		border-top: 2px solid #808080;
		border-left: 2px solid #808080;
		border-right: 2px solid #ffffff;
		border-bottom: 2px solid #ffffff;
	}

	.lb-table th {
		background-color: #c0c0c0;
		padding: 2px 8px;
		text-align: left;
		font-weight: bold;
		border-bottom: 1px solid #808080;
	}

	.lb-table td {
		padding: 2px 8px;
	}

	.lb-table tr:nth-child(even) td {
		background-color: #d4d4d4;
	}

	.clear-btn {
		font-family: "MS Sans Serif", Arial, sans-serif;
		font-size: 0.8rem;
		background-color: #c0c0c0;
		border-top: 2px solid #ffffff;
		border-left: 2px solid #ffffff;
		border-right: 2px solid #808080;
		border-bottom: 2px solid #808080;
		padding: 3px 10px;
		cursor: pointer;
		align-self: flex-end;
	}

	.clear-btn:active {
		border-top: 2px solid #808080;
		border-left: 2px solid #808080;
		border-right: 2px solid #ffffff;
		border-bottom: 2px solid #ffffff;
	}
</style>
