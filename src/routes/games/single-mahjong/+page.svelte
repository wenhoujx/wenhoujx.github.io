<!-- Mahjong.svelte -->
<script>
    import { onMount } from "svelte";
    import _ from "lodash";

    let tiles = [];
    let playerHand = [];
    let discards = [];
    let nSets = 4;
    let pongDisabled = true;
    let kongDisabled = true;
    let turn = 0;


    function reset() {
        tiles = allTiles();
        const n = 2 + nSets * 3;
        playerHand = sortHand(_.take(tiles, n));
        tiles = _.drop(tiles, n);
        discards = [];
    }

    function allTiles() {
        const range = (start, end) => {
            return Array(end - start + 1)
                .fill()
                .map((_, idx) => start + idx);
        };
        const dup = (arr, times) => {
            const res = [];

            for (let i = 0; i < times; i++) {
                res.push(...arr);
            }
            return res;
        };
        return shuffle([
            ...dup(
                range(1, 9).map((i) => `${i}b`), // bamboo
                4,
            ),
            ...dup(
                range(1, 9).map((i) => `${i}c`), // character
                4,
            ),
            ...dup(
                range(1, 9).map((i) => `${i}d`), // dot,
                4,
            ),
            ...dup(
                range(1, 7).map((i) => `${i}h`),
                4,
            ),
        ]);
        // console.log(JSON.stringify(tiles));
        // "1h", // east
        // "2h", // south
        // "3h", // west
        // "4h", // north
        // "5h", // center, china
        // "6h", // fa, fortune
        // "7h", // blank
    }

    function shuffle(array) {
        for (let i = array.length - 1; i > 0; i--) {
            const j = Math.floor(Math.random() * (i + 1));
            [array[i], array[j]] = [array[j], array[i]];
        }
        return array;
    }

    function sortHand(arr) {
        return arr.sort((a, b) => {
            // Extract letters
            let letterA = a.match(/[a-z]/)[0];
            let letterB = b.match(/[a-z]/)[0];

            // Compare letters
            if (letterA < letterB) return -1;
            if (letterA > letterB) return 1;

            // If letters are the same, compare numbers
            let numberA = parseInt(a.match(/\d+/)[0], 10);
            let numberB = parseInt(b.match(/\d+/)[0], 10);

            return numberA - numberB;
        });
    }
    function discardTile(tile) {
        let filtered = false;
        playerHand = _.filter(playerHand, (item) => {
            if (filtered) {
                return true;
            }
            if (item != tile) {
                return true;
            } else {
                filtered = true;
                return false;
            }
        });
        discards = [...discards, tile];
        turn = 1;
    }
</script>

<div>
    <p>select number of sets</p>
    {#each [1, 2, 3, 4] as option}
        <label style="display: inline-block; margin-right: 10px;">
            <input
                type="radio"
                name="radioGroup"
                value={option}
                bind:group={nSets}
                on:click={reset}
            />
            {option}
        </label>
    {/each}
</div>

<div class="hand">
    <h2>discards:</h2>
    {#each discards as d}
        <div class="tile">{d}</div>
    {/each}
</div>

<div class="hand">
    <h2>Your Hand: {_.size(playerHand)}</h2>
    {#each playerHand as tile}
        <div class="tile" on:click={() => discardTile(tile)}>{tile}</div>
    {/each}
</div>
<div class="control">
    <button disabled={turn === 0}>pass</button>
    <button disabled={pongDisabled}>pong</button>
    <button disabled={kongDisabled}>kong</button>
</div>

<style>
    .tile {
        display: inline-block;
        margin: 5px;
        padding: 10px;
        border: 1px solid #000;
        border-radius: 5px;
    }
    .tile:hover {
        background-color: greenyellow;
    }

    .hand,
    .discarded {
        margin: 10px 0;
    }
</style>
