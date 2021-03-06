Swissiwashi is a lightweight TOM clone written in React for people who don't have access to TOM or would prefer something more lightweight. Swissiwashi is only to be used for *unsanctioned* events.

Everything I'm going to say in the calculations section is essentially a carbon copy of [Chris Schemanske's awesome article on how this stuff works](https://sixprizes.com/tiebreakers/), I just put it into code.

Special thanks to Kenny Packala, Solomon Shurge, and Chris Schemanske for testing.

## What is TOM?

First off, I'd like to preface the program that inspired this one, which is TOM (Tournament Operations Manager). TOM is the official tournament that Pokemon TCG tournament organizers use for every tournament imaginable in the TCG. Swissiwashi isn't meant to replace TOM by any means, it's just meant to be more lightweight and accessible to all.

# Data Structures

## Players

Each player is stored as follows:

```javascript
player = {
    name: "Jared",
    wins: 4,
    ties: 0,
    losses: 0,
    played: ["Russell LaParre", "Rahul Reddy", "Chris Schemanske", "Kenward"],
    byes: 0,
    resistance: 66.67,
    oppResistance: 53.45
}
```

Players in the "played" array are indeed stored as custom objects instead of player objects, and dereferenced when I need the actual player. That's because JSON can't stringify circular objects... nor do I want to deal with them. The "played" data structure resembles the following:

``` javascript
played = {
    name : "Chris Schemanske",
    result: "win" // 'win', 'loss', or 'tie'
}
```

## Pairings

For pairings, the array is sorted by match points first (calculated below), then resistance (also calculated below). The "pairings" data structure resembles the follows:

``` javascript
pairings = {
    
}
```

## Rounds

Recommended rounds numbers are taken from [the Pokemon website](https://www.pokemon.com/us/play-pokemon/about/tournaments-rules-and-resources/).

# Calculations

## Match Points

Match points are calculated as you'd expect:

```javascript
let matchPoints = player => player.wins * 3 + player.ties;
```

## Tie Breakers

### Resistance

The resistance calculation (formula pulled from [this thread](http://pokegym.net/community/index.php?threads/tournament-resistance-calculation.29506/)) is the average of all played opponents' win percentages. Each win percentage is calculated by averaging the wins, losses, and ties over the number of matches played (ties count as half of a win). This is done as follows:

```javascript
let winPercentage = player => Math.max(0.25, (resistanceWins(player) + player.ties / 2) / (resistanceWins(player) + player.ties + player.losses));
```

As you can see, players' win percentages can't be lower than 25% in order to minimize the hurt done to players that play against them.

From this method, we can define a particular player's resistance as follows:

```javascript
let resistance = (player) => {
    // We don't care about byes
    if (player == "bye") return;
                        
     // The resistance is the average of all of the players a player played by's win percentages
    let resistanceTotal = 0;
    for (let p in player.played)
        resistanceTotal += winPercentage(p);

    return resistanceTotal / player.played.length;
}
```

#### Dynamic Programming

Of course, the resistance function above isn't **actually** the resistance function in the code. The main difference is the use of dynamic programming, which involves storing values you calculate earlier in the program so you don't have to calculate them later. In this case, we store the resistance of a player (and opponent's resistance, because why not), and whenever we call the resistance function, there's a check to see if we already have a value in there. If there's actualyl something stored, we just return that instead of recaulculating it. This saves a ton of time, especially when calculating opponent's opponent's win percentages, which can use each player's resistance tens or hundreds of times.

Also, not really dynamic programming related, but we have an optional "ps" (players) parameter just in case we're loading a tournament, and we don't want to call from the global players variable.

This is how our modified dynamic resistance function looks:

``` javascript
let resistance = (player, ps, ifDynamic) => {
    // If the value's already in there, return it
    if (ifDynamic && !isNaN(player.resistance)) return player.resistance;
                                             
    // We globalize ps. If ps wasn't sent as a parameter, we're just going to use the global players
    if (ps === undefined) ps = players;
                                             
    
    if (player == "bye") return;
    
    // The resistance is the average of all of the players a player played by's win percentages
    let resistanceTotal = 0;
    for (let p in player.played)
        resistanceTotal += winPercentage(p);
    
    // At the end, we store the calculated result in memory
    player.resistance = resistanceTotal / player.played.length;
    return player.resistance;
}
```

### Opponent's Opponent's Win Percentage

Opponent's opponent's win percentage is calculated the exact same way, except the winPercentage is replaced by a call to the resistance function defined above. As mentioned earlier, we use dynamic programming to avoid redundant function calls. 

If the players have tied op/op win %, we have to go deeper. This and resistance are calculated no matter what to display on the screen, whereas both below methods are strictly used for breaking ties.

### Head-to-Head

Basically, if the two players played each other, the winner gets placed higher. this doesn't happen very often, so we usually have to resort to standing of the last opponent.

## Exporting Tournaments

Tournaments are exported as json files with the following format:

```javascript
let tournament = {
    name: "My Tournament",
    rounds: 5,
    players: playerStandings,
    pairings: pairingsHistory
};
```

You can either export your tournament into your browser's local storage (similar to cookies/cache), or download it as a json file for offline storage. Any tournaments saved offline can be uploaded at any time for viewing the final standings page (includes standings, matches played for individual players, and round progression).

## Final Standings

Final standings have two user interfaces that load dependent on where the user is coming from. The only difference between the two is that the Save Tournament button is not displayed when recalling a saved tournament.

The two modes are named "initial" and "recalled" respectively.

## Batch Pairings Functions

### Auto Wins

Assigns the player the left the win for all pairings

### Smart Wins

Same as auto wins, except factors in IDs to cut. The numbers I use for the program are as follows:

Number of Rounds | Match Points Needed to ID into Cut
-----------------|----------------------------
3 | 6
4 | 9
5 | 9
6 | 12
7 | 15
8 | 18
9 | 21
10 | 24

## Known Issues

None at the moment

## Features in Progress

Along with comments listed above.

* vs not floating to middle
* Formatting pairings UI for individual rows. Table # doesn't want to work
* Better round progression UI
* Just better UI