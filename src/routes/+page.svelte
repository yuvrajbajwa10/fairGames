<script lang="ts">
	import { onMount } from 'svelte';
	import { get, writable, type Writable } from 'svelte/store';

	var teamSizeValue: number = 2;
	var courtSizeValue: number = 2;
	let textarea: HTMLTextAreaElement;

	function autoResize() {
		textarea.style.height = 'auto';
		textarea.style.height = textarea?.scrollHeight + 20 + 'px';
	}

	type team = {
		players: string[];
		consecutiveWins: number;
	};

	type match = {
		team1: team;
		team2: team;
		court: number;
		round: number;
		winner: 'team1' | 'team2' | null;
	};
	type player = {
		name: string;
		playTime: number;
	};

	let players: string[] = [];
	let matches: Writable<match[]> = writable([]);
	let currentRound: number = 1;
	let playersPlayTime: player[] = [];
	// cache the value of teamSizeValue from the form
	const teamPlayerSize: Writable<number> = writable(teamSizeValue);
	const courtSize: Writable<number> = writable(courtSizeValue);
	const calculate = () => {
		teamPlayerSize.set(teamSizeValue);
		courtSize.set(courtSizeValue);
		players = textarea.value.split('\n').filter((player) => player.trim() !== '');
		players.sort(() => Math.random() - 0.5);
		// Initialize play time for each player
		playersPlayTime = players.map((player) => ({ name: player, playTime: 0 }));

		matches.set([]);
		currentRound = 1;
		generateNextMatches();
	};
	const bubbleSort = (arr: any[], compare: (a: any, b: any) => number): any[] => {
		let len = arr.length;
		for (let i = 0; i < len - 1; i++) {
			for (let j = 0; j < len - i - 1; j++) {
				if (compare(arr[j], arr[j + 1]) > 0) {
					swap(arr, j, j + 1);
				}
			}
		}
		return arr;
	};
	const swap = (arr: any[], i: number, j: number) => {
		[arr[i], arr[j]] = [arr[j], arr[i]];
	};
	const slidinWindowSort = (arr: any[], compare: (a: any, b: any) => number): any[] => {
		let len = arr.length;
		let windowSize = 2;
		for (let i = 0; i < len - 1; i += windowSize) {
			let end = Math.min(i + windowSize, len);
			let sorted = bubbleSort(arr.slice(i, end), compare);
			for (let j = i; j < end; j++) {
				arr[j] = sorted[j - i];
			}
		}
		return arr;
	};

	const generateNextMatches = (winnerTeams?: team[]) => {
		let sortedPlayers = [...playersPlayTime].sort((a, b) => a.playTime - b.playTime);
		let courtsInUse = new Set(
			get(matches)
				.filter((m) => m.round === currentRound)
				.map((m) => m.court)
		);
		console.log(sortedPlayers);

		// Sort winner teams by consecutive wins (descending)
		winnerTeams = slidinWindowSort(winnerTeams || [], (a: team, b: team) => {
			if (a.consecutiveWins === b.consecutiveWins) {
				return 0;
			}
			if (a.consecutiveWins > b.consecutiveWins) {
				return -1;
			}
			return 1;
		});

		while (sortedPlayers.length >= $teamPlayerSize * 2 && courtsInUse.size < $courtSize) {
			const team1Players: string[] = [];
			const team2Players: string[] = [];
			let selectedPlayers: string[] = [];
			let team1ConsecutiveWins = 0;
			let team2ConsecutiveWins = 0;

			function addPlayerToTeam(team: string[], playerName: string) {
				if (selectedPlayers.includes(playerName)) {
					return;
				}
				team.push(playerName);
				selectedPlayers.push(playerName);
				updatePlayerPlayTime(playerName);
				sortedPlayers = sortedPlayers.filter((p) => p.name !== playerName);
			}

			function updatePlayerPlayTime(playerName: string) {
				const index = playersPlayTime.findIndex((p) => p.name === playerName);
				playersPlayTime[index].playTime++;
			}

			// Handle winner teams
			if (winnerTeams && winnerTeams.length > 0) {
				const winnerTeam = winnerTeams.shift();
				if (winnerTeam && winnerTeam.consecutiveWins < 2) {
					winnerTeam.players.forEach((playerName) => {
						if (sortedPlayers.some((p) => p.name === playerName)) {
							addPlayerToTeam(team1Players, playerName);
						}
					});
					team1ConsecutiveWins = winnerTeam.consecutiveWins;
				}
			}

			// Fill remaining spots
			for (let player of sortedPlayers) {
				if (team1Players.length < $teamPlayerSize) {
					addPlayerToTeam(team1Players, player.name);
				} else if (team2Players.length < $teamPlayerSize) {
					addPlayerToTeam(team2Players, player.name);
				}

				if (team1Players.length === $teamPlayerSize && team2Players.length === $teamPlayerSize) {
					break;
				}
			}

			// If we don't have enough players for a full match, break the loop

			if (team1Players.length < $teamPlayerSize || team2Players.length < $teamPlayerSize) {
				break;
			}

			// Find an available court
			let court = 1;
			while (courtsInUse.has(court)) {
				court++;
			}
			// Create and add the new match
			matches.update((currentMatches) => [
				...currentMatches,
				{
					team1: {
						players: team1Players,
						consecutiveWins: team1ConsecutiveWins
					},
					team2: {
						players: team2Players,
						consecutiveWins: team2ConsecutiveWins
					},
					court: court,
					round: currentRound,
					winner: null
				}
			]);
		}
	};
	const togglePrevMatchs: Writable<boolean> = writable(false);

	const updateMatch = (matchIndex: number, winner: 'team1' | 'team2') => {
		const match = get(matches)[matchIndex];
		// Mark the match as completed
		match.winner = winner;

		// Generate next match if all current round matches are completed
		if (get(matches).filter((m) => m.round === currentRound && !m.winner).length === 0) {
			currentRound++;
			var winners = get(matches)
				.filter((m) => m.round === currentRound - 1)
				.filter((m) => m.winner)
				.map((m) => (m.winner === 'team1' ? m.team1 : m.team2));

			winners.forEach((winner) => {
				winner.consecutiveWins++;
			});
			generateNextMatches(winners);
		}
		matches.update((value) => [...value]);
	};

	onMount(() => {
		textarea.value = 'a\nb\nc\nd\ne';
		autoResize();
	});
</script>

<div class="flex justify-center flex-col [&>input]:border [&>textarea]:border space-y-1 *:p-2">
	<!-- label -->
	<label for="teamSize">Team Size</label>
	<input type="number" placeholder="team size" bind:value={teamSizeValue} />
	<label for="courtSize">Amount of Court</label>
	<input type="number" placeholder="court size" bind:value={courtSizeValue} />
	<label for="players">
		Players
		<span class="text-sm font-light">- one player per line</span>
	</label>
	<textarea bind:this={textarea} on:input={autoResize} class="w-full"></textarea>
	<button class="bg-blue-500 text-blue-50" on:click={() => calculate()}>Generate</button>
	<div>
		<div class="flex justify-end">
			{#if get(matches).length > 0}
				<button
					class={`${$togglePrevMatchs ? 'bg-blue-500' : 'bg-red-500'} text-blue-50 p-2 rounded-lg`}
					on:click={() => {
						togglePrevMatchs.update((value) => !value);
					}}
				>
					Toggle Previous Matches
				</button>
			{/if}
		</div>
		<table class="w-full">
			<thead>
				<tr>
					<th>Match</th>
					<th>Round</th>
					<th>Team 1</th>
					<th>Team 2</th>
					{#if $courtSize > 1}
						<th>Court</th>
					{/if}
					<th>Team 1 Win</th>
					<th>Team 2 Win</th>
				</tr>
			</thead>
			<tbody>
				{#each $matches as match, index}
					{#if ($togglePrevMatchs && match.winner == null) || !$togglePrevMatchs}
						{#key index}
							<tr class={match.round % 2 === 0 ? 'bg-gray-100' : ''}>
								<td>{index + 1}</td>
								<td>{match.round}</td>
								<td
									class={match.winner === 'team1'
										? 'bg-green-200'
										: match.winner === 'team2'
											? 'bg-red-200'
											: ''}
								>
									{match.team1.players.join(', ')}
								</td>
								<td
									class={match.winner === 'team2'
										? 'bg-green-200'
										: match.winner === 'team1'
											? 'bg-red-200'
											: ''}
								>
									{match.team2.players.join(', ')}
								</td>
								{#if $courtSize > 1}
									<td>{match.court}</td>
								{/if}
								<td>
									<input
										id="radioteam1${index}"
										type="radio"
										name="winner${index}"
										disabled={match.winner !== null}
										checked={match.winner === 'team1'}
										on:change={() => updateMatch(index, 'team1')}
									/>
								</td>
								<td>
									<input
										id="radioteam2${index}"
										type="radio"
										name="winner${index}"
										disabled={match.winner !== null}
										checked={match.winner === 'team2'}
										on:change={() => updateMatch(index, 'team2')}
									/>
								</td>
							</tr>
						{/key}
					{/if}
				{/each}
			</tbody>
		</table>
	</div>
</div>
