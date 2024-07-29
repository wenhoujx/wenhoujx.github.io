<!-- Mahjong.svelte -->
<script>
    import { onMount } from "svelte";
    import _ from "lodash";

    let allTiles = [];
    let hand = [];
    let discards = [];
    let nSets = 4;

    let turn = 0;

    $: {
        if (turn == 1 || turn == 2 || turn == 3) {
            otherPlayerPlay();
        }
    }

    const takeLast = (arr) => {
        const ele = _.last(arr);
        return [_.dropRight(arr, 1), ele];
    };

    function otherPlayerPlay() {
        let tile;
        [allTiles, tile] = takeLast(allTiles);
        appendToDiscards(tile);
    }

    onMount(() => {
        reset();
    });

    function reset() {
        allTiles = shuffleAllTiles();
        const n = 2 + nSets * 3;
        hand = sortHand(_.take(allTiles, n));
        allTiles = _.drop(allTiles, n);
        discards = [];
    }

    function shuffleAllTiles() {
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
        // console.log(JSON.stringify(arr));
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
        if (turn != 0) {
            return;
        }
        let filtered = false;
        hand = _.filter(hand, (item) => {
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

        appendToDiscards(tile);
        moveOn();
    }
    function appendToDiscards(tile) {
        discards = _.takeRight([...discards, tile], 8);
    }
    function moveOn() {
        turn = (turn + 1) % 4;
    }
    function takeDiscarded() {
        let tile;
        [discards, tile] = takeLast(discards);
        // console.log(tile);
        hand = sortHand([...hand, tile]);
    }
    function draw() {
        let tile;
        [allTiles, tile] = takeLast(allTiles);
        hand = sortHand([...hand, tile]);
    }
    function hasPong() {
        const tile = _.last(discards);
        const count = _.size(_.filter(hand, (el) => el === tile));
        return count == 2;
    }
    function hasKong() {
        const tile = _.last(discards);
        const count = _.size(_.filter(hand, (el) => el === tile));
        return count == 3;
    }
    function takePong() {
        if (!hasPong()) {
            return;
        }
        let tile;
        [discards, tile] = takeLast(discards);
        hand = sortHand([...hand, tile]);
    }
    function takeKong() {
        if (!hasKong()) {
            return;
        }
        let tile;
        [discards, tile] = takeLast(discards);
        hand = sortHand([...hand, tile]);
    }
</script>

<div class="container">
    <div>
        <p>select number of sets</p>
        {#each [1, 2, 3, 4] as option}
            <label style="display: inline-block; margin-right: 10px;">
                <input
                    type="radio"
                    name="radioGroup"
                    value={option}
                    bind:group={nSets}
                    on:change={reset}
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
        <h2>Your Hand: {_.size(hand)}</h2>
        {#each hand as tile}
            <div class="tile" on:click={() => discardTile(tile)}>{tile}</div>
        {/each}
    </div>
    <div class="control">
        {#if turn == 0}
            <div>it's your turn, click a tile to play</div>
        {:else}
            <div>player {turn + 1} played {_.last(discards)}</div>
        {/if}
        <button disabled={turn === 0 || turn == 3} on:click={moveOn}
            >pass</button
        >
        <button
            disabled={turn != 3}
            on:click={() => {
                takeDiscarded();
                moveOn();
            }}>take</button
        >
        <button
            disabled={turn != 3}
            on:click={() => {
                draw();
                moveOn();
            }}>draw</button
        >
        <button disabled={turn == 0 || !hasPong()} on:click={takePong}
            >pong</button
        >
        <button disabled={turn == 0 || !hasKong()} on:click={takeKong}
            >kong</button
        >
    </div>

    <div>
        <div>remaining {_.size(allTiles)}</div>
        <div>hand {JSON.stringify(hand)}</div>
        <div>discards {JSON.stringify(discards)}</div>
    </div>
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
