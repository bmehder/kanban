<script>
  import Card from '../UI/Card.svelte';
  import Task from '../UI/Task.svelte';
  import Flex from '../Layout/Flex.svelte';
  import { flip } from 'svelte/animate';
  import { fade } from 'svelte/transition';
  import { onMount, tick } from 'svelte';
  import { kanbanBoard } from '../Stores/kanbanStore.js';

  let newCardCreated = false;
  let hideAddCardBtn = false;
  let newCardTitle = '';
  let newCardInput;
  let updateDBTimeout;
  let boardInitialized = false;
  const databaseURL =
    'https://kanban-eed3c-default-rtdb.firebaseio.com/kanban-board.json';
  const boardDefaultValue = [
    {
      id: 0,
      name: 'To Do',
      tasks: [
        { id: 0, value: 'Task 2' },
        { id: 1, value: 'Task 3' },
      ],
    },
    {
      id: 1,
      name: 'Doing',
      tasks: [{ id: 2, value: 'Task 1' }],
    },
    {
      id: 2,
      name: 'Testing',
      tasks: [],
    },
    {
      id: 3,
      name: 'Done',
      tasks: [],
    },
  ];
  let loadingMsg = 'Initializing...';

  onMount(() => {
    (async () => {
      try {
        let response = await fetch(databaseURL);
        let data = await response.json();
        if (!data) {
          data = boardDefaultValue;
          initializeDB(boardDefaultValue);
        }
        data.forEach((element) => {
          if (!element.hasOwnProperty('tasks')) {
            element['tasks'] = [];
          }
        });
        kanbanBoard.initialize(data);
        boardInitialized = true;
      } catch (error) {
        loadingMsg = 'Database communication error. Loading default board...';
        setTimeout(() => {
          kanbanBoard.initialize(boardDefaultValue);
          boardInitialized = true;
        }, 2000);
      }
    })();
  });

  $: {
    clearTimeout(updateDBTimeout);
    updateDBTimeout = setTimeout(() => {
      (async function updateDB() {
        let response = await fetch(databaseURL, {
          method: 'PUT',
          body: JSON.stringify($kanbanBoard),
          headers: { 'Content-Type': 'application/json' },
        });
      })();
    }, 5000);
  }

  // -------------- //
  // Card functions //
  // -------------- //
  function cardIsNotEmpty(cardInd) {
    if (
      !($kanbanBoard[cardInd].tasks.slice(-1) == '') ||
      $kanbanBoard[cardInd].tasks.length === 0
    ) {
      return true;
    } else {
      return false;
    }
  }

  function addCard() {
    newCardCreated = true;
    hideAddCardBtn = true;
    (async () => {
      await tick();
      newCardInput.focus();
    })();
  }

  function newCardAdded(e) {
    newCardTitle = '';
    newCardCreated = false;
    hideAddCardBtn = false;
    const newCardId = findIdForNewCard();
    const newTaskId = findIdForNewTask();
    if (e.target.value != '') {
      kanbanBoard.addCard(newCardId, e.target.value, newTaskId);
    }
  }

  function deleteCard(e) {
    hideAddCardBtn = true;
    const cardId = parseInt(e.detail.slice(2));
    kanbanBoard.deleteCard(cardId);
    setTimeout(() => {
      hideAddCardBtn = false;
    }, 800);
  }

  function findIdForNewCard() {
    const cardIDs = $kanbanBoard.map((o) => o.id);
    const newCardId = maxValue(cardIDs) + 1;
    return newCardId ? newCardId : 0;
  }

  // -------------- //
  // Task functions //
  // -------------- //
  function addTask(e, targetValue = '') {
    const cardInd = e.target.id.slice(8);
    const newTaskId = findIdForNewTask();
    if (cardIsNotEmpty(cardInd)) {
      kanbanBoard.addTask(cardInd, newTaskId, targetValue);
    }
  }

  function deleteTask(e) {
    const taskID = parseInt(e.detail.slice(2));
    const cardIndex = findCardIndexOfTaskID(e.detail);
    kanbanBoard.deleteTask(cardIndex, taskID);
  }

  function moveTaskLeft(t) {
    const e = { detail: t.detail.id };
    deleteTask(e);
    const taskId = parseInt(t.detail.id.slice(2));
    const cardIndex = findCardIndexOfTaskID(e.detail);
    const cardToAdd = { target: { id: 'addTask_' + parseInt(cardIndex - 1) } };
    addTask(cardToAdd, t.detail.value);
  }

  function moveTaskRight(t) {
    const e = { detail: t.detail.id };
    deleteTask(e);
    const taskId = parseInt(t.detail.id.slice(2));
    const cardIndex = findCardIndexOfTaskID(e.detail);
    const cardToAdd = { target: { id: 'addTask_' + parseInt(cardIndex + 1) } };
    addTask(cardToAdd, t.detail.value);
  }

  function taskBlurred(t) {
    const e = { detail: t.detail.id };
    t.detail.value === '' ? deleteTask(e) : '';
  }

  function findIdForNewTask() {
    let taskIDsArray = [];
    let taskIDs = [];
    $kanbanBoard.forEach((b) => {
      taskIDs = b.tasks.map((t) => t.id);
      taskIDsArray = [...taskIDsArray, taskIDs];
    });
    const newTaskId = maxValue(taskIDsArray.flat()) + 1;
    return newTaskId ? newTaskId : 0;
  }

  function findCardIndexOfTaskID(taskID) {
    const cardID = parseInt(
      document.getElementById(taskID).parentNode.parentNode.id.slice(2)
    );
    return $kanbanBoard.findIndex((b) => b.id === cardID);
  }

  // -------------------- //
  // Additional functions //
  // -------------------- //
  function maxValue(arr) {
    let max = arr[0];
    for (let val of arr) {
      if (val > max) {
        max = val;
      }
    }
    return max;
  }

  async function initializeDB(value) {
    let response = await fetch(databaseURL, {
      method: 'PUT',
      body: JSON.stringify(value),
      headers: { 'Content-Type': 'application/json' },
    });
  }
</script>

<style>
  h3 {
    margin: 2rem 2rem 0 0;
  }
  button {
    margin: 1.5rem auto 0.5rem auto;
    padding: 0.1rem 0.5rem;
    color: rgb(0, 121, 36);
    background-color: #f4f4f4;
  }
  button:not(:disabled):active {
    background-color: rgb(0, 121, 36);
    color: #f4f4f4;
  }
  input {
    width: 95%;
    margin: 0.5rem 0.3rem;
    padding: 0.2rem;
    background: white;
    color: #333;
    border-radius: 0.2rem;
  }
  .newCard {
    margin: 0.5rem;
    padding: 4rem 1rem;
    background-color: rgb(47, 10, 104);
    color: white;
    font-weight: bold;
  }

  #arrow {
    text-align: right;
    font-size: 5rem;
    font-weight: bold;
    line-height: 0.1rem;
  }
</style>

{#if !boardInitialized}
  <Flex>
    <h3>{loadingMsg}</h3>
  </Flex>
{:else}
  <Flex noWrap={true} align={'start'}>
    {#each $kanbanBoard as card, i (card.id)}
      <section animate:flip={{ delay: 400, duration: 400 }}>
        <Card id={'c_' + card.id} title={card.name} on:deleteCard={deleteCard}>
          {#each card.tasks as task, t (task.id)}
            <Task
              id={'t_' + task.id}
              firstCard={i === 0 || $kanbanBoard[i].tasks[t].value == ''}
              lastCard={i === $kanbanBoard.length - 1 || $kanbanBoard[i].tasks[t].value == ''}
              bind:value={$kanbanBoard[i].tasks[t].value}
              on:deleteTask={deleteTask}
              on:moveLeft={moveTaskLeft}
              on:moveRight={moveTaskRight}
              on:blurred={taskBlurred} />
          {/each}
          <Flex><button id={'addTask_' + i} on:click={addTask}>+</button></Flex>
        </Card>
      </section>
    {:else}
      <section>
        <h3>Your board is empty</h3>
        <p>Start adding lists with this button!</p>
        <div id="arrow">→</div>
      </section>
    {/each}
    {#if newCardCreated}
      <Card id="newCard" title={newCardTitle} newCard={true}>
        <input
          type="text"
          placeholder="Insert list name..."
          bind:value={newCardTitle}
          bind:this={newCardInput}
          on:blur={newCardAdded} />
      </Card>
    {/if}
    {#if !hideAddCardBtn}
      <section in:fade>
        <button class="newCard" on:click={addCard}>+</button>
      </section>
    {/if}
  </Flex>
{/if}
