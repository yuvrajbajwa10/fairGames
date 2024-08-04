<script lang="ts">
	import { onMount } from 'svelte';
	import { get, writable, type Writable } from 'svelte/store';

	var teamSizeValue: number = 2;
	var courtSizeValue: number = 1;
	let textarea: HTMLTextAreaElement;

	function autoResize() {
		textarea.style.height = 'auto';
		textarea.style.height = textarea?.scrollHeight + 20 + 'px';
	}

	type team = {
		players: string[];
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
	let playersPlayTime: Writable<player[]> = writable([]);
	const teamPlayerSize: Writable<number> = writable(teamSizeValue);
	const courtSize: Writable<number> = writable(courtSizeValue);

	function saveToLocalStorage() {
		localStorage.setItem('players', JSON.stringify(get(playersPlayTime)));
		localStorage.setItem('matches', JSON.stringify(get(matches)));
		localStorage.setItem('currentRound', currentRound.toString());
		localStorage.setItem('teamSize', get(teamPlayerSize).toString());
		localStorage.setItem('courtSize', get(courtSize).toString());
		localStorage.setItem('playersList', textarea.value);
	}

	function loadFromLocalStorage() {
		const savedPlayers = localStorage.getItem('players');
		const savedMatches = localStorage.getItem('matches');
		const savedCurrentRound = localStorage.getItem('currentRound');
		const savedTeamSize = localStorage.getItem('teamSize');
		const savedCourtSize = localStorage.getItem('courtSize');
		const savedPlayersList = localStorage.getItem('playersList');

		if (savedPlayers) playersPlayTime.set(JSON.parse(savedPlayers));
		if (savedMatches) matches.set(JSON.parse(savedMatches));
		if (savedCurrentRound) currentRound = parseInt(savedCurrentRound);
		if (savedTeamSize) {
			teamPlayerSize.set(parseInt(savedTeamSize));
			teamSizeValue = parseInt(savedTeamSize);
		}
		if (savedCourtSize) {
			courtSize.set(parseInt(savedCourtSize));
			courtSizeValue = parseInt(savedCourtSize);
		}
		if (savedPlayersList) textarea.value = savedPlayersList;
	}

	const calculate = () => {
		teamPlayerSize.set(teamSizeValue);
		courtSize.set(courtSizeValue);
		players = textarea.value.split('\n').filter((player) => player.trim() !== '');
		players.sort(() => Math.random() - 0.5);
		$playersPlayTime = players.map((player) => ({ name: player, playTime: 1 }));

		matches.set([]);
		currentRound = 1;
		generateNextMatches();
		saveToLocalStorage();
	};

	const generateNextMatches = () => {
		playersPlayTime.update((players) => {
			let sortedPlayers = [...players].sort((a, b) => {
				if (a.playTime === b.playTime) {
					return Math.random() - 0.5;
				} else {
					return a.playTime - b.playTime;
				}
			});

			let courtsInUse = new Set();
			currentRound = 1;

			let existingTeams: { [key: string]: string[] } = {};

			while (sortedPlayers.length >= get(teamPlayerSize) * 2 && courtsInUse.size < get(courtSize)) {
				let team1Players: string[] = [];
				let team2Players: string[] = [];

				for (let teamKey in existingTeams) {
					if (
						existingTeams[teamKey].every((player) => sortedPlayers.some((p) => p.name === player))
					) {
						if (team1Players.length === 0) {
							team1Players = existingTeams[teamKey];
						} else if (team2Players.length === 0) {
							team2Players = existingTeams[teamKey];
						}
						if (team1Players.length > 0 && team2Players.length > 0) break;
					}
				}

				while (
					team1Players.length < get(teamPlayerSize) ||
					team2Players.length < get(teamPlayerSize)
				) {
					const player = sortedPlayers.shift();
					if (player) {
						if (team1Players.length < get(teamPlayerSize)) {
							team1Players.push(player.name);
						} else {
							team2Players.push(player.name);
						}
						const index = players.findIndex((p) => p.name === player.name);
						players[index].playTime++;
					}
				}

				existingTeams[team1Players.sort().join(',')] = team1Players;
				existingTeams[team2Players.sort().join(',')] = team2Players;

				let court = 1;
				while (courtsInUse.has(court)) {
					court++;
				}

				matches.update((currentMatches) => [
					...currentMatches,
					{
						team1: { players: team1Players },
						team2: { players: team2Players },
						court: court,
						round: currentRound,
						winner: null
					}
				]);

				courtsInUse.add(court);
				currentRound++;

				sortedPlayers = sortedPlayers.filter(
					(p) => !team1Players.includes(p.name) && !team2Players.includes(p.name)
				);
			}

			return players;
		});
		saveToLocalStorage();
	};

	const togglePrevMatchs: Writable<boolean> = writable(false);

	const updateMatch = (matchIndex: number, winner: 'team1' | 'team2') => {
		const match = get(matches)[matchIndex];
		match.winner = winner;

		if (get(matches).filter((m) => m.round === currentRound && !m.winner).length === 0) {
			currentRound++;
			generateNextMatches();
		}
		matches.update((value) => [...value]);
		saveToLocalStorage();
	};

	function resetGame() {
		localStorage.clear();
		players = [];
		matches.set([]);
		currentRound = 1;
		playersPlayTime.set([]);
		teamPlayerSize.set(2);
		courtSize.set(1);
		teamSizeValue = 2;
		courtSizeValue = 1;
		textarea.value = '';
	}

	onMount(() => {
		loadFromLocalStorage();
		if (!textarea.value) {
			textarea.value = 'yash\nyuvi\npaul\nivor\nkishan';
		}
		autoResize();
	});
</script>

<div class="flex justify-center flex-col [&>input]:border [&>textarea]:border space-y-1 *:p-2">
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
	<button class="bg-red-500 text-red-50" on:click={() => resetGame()}>Reset Game</button>
	<div>
		<div class="flex justify-end">
			{#if $matches.length > 0}
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
