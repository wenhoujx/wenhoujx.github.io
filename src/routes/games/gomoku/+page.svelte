<script>
    import { onMount } from "svelte";

    let board = [];
    let history = [];
    let currentPlayer = "X";
    let winner = null;

    const BOARD_SIZE = 25;

    function initializeBoard() {
        board = Array(BOARD_SIZE)
            .fill()
            .map(() => Array(BOARD_SIZE).fill(" "));
    }

    function handleCellClick(row, col) {
        if (board[row][col] === " " && !winner) {
            board[row][col] = currentPlayer;
            history.push(JSON.parse(JSON.stringify(board)));
            if (checkWin(row, col, currentPlayer)) {
                winner = currentPlayer;
            }
            currentPlayer = currentPlayer === "X" ? "O" : "X";
        }
    }

    function rollback() {
        if (history.length > 1) {
            history.pop();
            board = JSON.parse(JSON.stringify(history[history.length - 1]));
            winner = null;
            currentPlayer = currentPlayer === "X" ? "O" : "X";
        }
    }

    function checkWin(row, col, player) {
        return (
            checkDirection(row, col, 1, 0, player) || // Horizontal
            checkDirection(row, col, 0, 1, player) || // Vertical
            checkDirection(row, col, 1, 1, player) || // Diagonal down-right
            checkDirection(row, col, 1, -1, player)
        ); // Diagonal down-left
    }

    function checkDirection(row, col, rowDir, colDir, player) {
        let count = 1;
        for (let i = 1; i < 5; i++) {
            const r = row + i * rowDir;
            const c = col + i * colDir;
            if (
                r < 0 ||
                r >= BOARD_SIZE ||
                c < 0 ||
                c >= BOARD_SIZE ||
                board[r][c] !== player
            )
                break;
            count++;
        }
        for (let i = 1; i < 5; i++) {
            const r = row - i * rowDir;
            const c = col - i * colDir;
            if (
                r < 0 ||
                r >= BOARD_SIZE ||
                c < 0 ||
                c >= BOARD_SIZE ||
                board[r][c] !== player
            )
                break;
            count++;
        }
        return count >= 5;
    }

    onMount(() => {
        initializeBoard();
        history.push(JSON.parse(JSON.stringify(board)));
    });
</script>

<div>
    {#if winner}
        <div class="winner {winner == 'X' ? 'X' : 'O'}">{winner} wins!</div>
    {/if}
    <div class="board">
        {#each board as row, rowIndex}
            {#each row as cell, colIndex}
                <div
                    class="cell"
                    on:click={() => handleCellClick(rowIndex, colIndex)}
                >
                    <div class={cell}>{cell}</div>
                </div>
            {/each}
        {/each}
    </div>
    <button on:click={rollback}>Rollback</button>
</div>

<style>
    .board {
        display: grid;
        grid-template-columns: repeat(25, 25px);
        grid-template-rows: repeat(25, 25px);
        gap: 1px;
        margin: 20px auto;
        width: max-content;
    }

    .cell {
        width: 25px;
        height: 25px;
        border: 1px solid #ccc;
        display: flex;
        justify-content: center;
        align-items: center;
        cursor: pointer;
        background-color: #fff;
    }

    .cell:hover {
        background-color: #f0f0f0;
    }

    .winner {
        text-align: center;
        font-size: 1.5em;
        margin: 20px;
    }

    button {
        display: block;
        margin: 20px auto;
    }
    .X {
        color: red;
        font-weight: bold;
    }

    .O {
        color: green;
        font-weight: bold;
    }
</style>
