<script>
// todo: add back in focus to all the correct fields.
import {onMount} from "svelte";

let users = [];
const hostname = "pokerstore.kayser.workers.dev";
function handleKeydown(event){
    if (event.keyCode == 38) {
      // up arrow
      chat_container.scrollBy(0, -50);
    } else if (event.keyCode == 40) {
      // down arrow
      chat_container.scrollBy(0, 50);
    } else if (event.keyCode == 33) {
      // page up
      chat_container.scrollBy(0, -chat_container.clientHeight + 50);
    } else if (event.keyCode == 34) {
      // page down
      chat_container.scrollBy(0, chat_container.clientHeight - 50);
    }
}

function submitMessage() {
  if (currentWebSocket) {
    currentWebSocket.send(JSON.stringify({message: message_text}));
    message_text = "";

    // Scroll to bottom whenever sending a message.
    chat_container.scroll({ top: chat_container.scrollHeight, behavior: 'smooth' });
  }
}

function startChat() {
  show_room_picker =false;
  show_chatroom = true;

  // Normalize the room name a bit.
  room_name = room_name.replace(/[^a-zA-Z0-9_-]/g, "").replace(/_/g, "-").toLowerCase();

  if (room_name.length > 32 && !room_name.match(/^[0-9a-f]{64}$/)) {
    addChatMessage("*", "Invalid room name.");
    return;
  }

  document.location.hash = "#" + room_name;
  join();
}
//
let lastSeenTimestamp = 0;
let wroteWelcomeMessages = false;
//
function join() {
  // If we are running via wrangler dev, use ws:
  const wss = document.location.protocol === "http:" ? "ws://" : "wss://";
  let ws = new WebSocket(wss + hostname + "/api/room/" + room_name + "/websocket");
  let rejoined = false;
  let startTime = Date.now();

  let rejoin = async () => {
    if (!rejoined) {
      rejoined = true;
      currentWebSocket = null;

      // Clear the roster.
        users = []

      // Don't try to reconnect too rapidly.
      let timeSinceLastJoin = Date.now() - startTime;
      if (timeSinceLastJoin < 10000) {
        // Less than 10 seconds elapsed since last join. Pause a bit.
        await new Promise(resolve => setTimeout(resolve, 10000 - timeSinceLastJoin));
      }

      // OK, reconnect now!
      join();
    }
  }

  ws.addEventListener("open", event => {
    currentWebSocket = ws;

    // Send user info message.
    ws.send(JSON.stringify({name: username}));
  });

  ws.addEventListener("message", event => {
    let data = JSON.parse(event.data);

    if (data.error) {
      addChatMessage("*", "Error: " + data.error);
    } else if (data.joined) {
        addChatMessage("*", data.joined + " joined.")
      users = [...users, data.joined]
    } else if (data.quit) {
        addChatMessage("*", data.quit + " left.")
        users = users.filter(user => user !== data.quit)
    } else if (data.ready) {
      // All pre-join messages have been delivered.
      if (!wroteWelcomeMessages) {
        wroteWelcomeMessages = true;
        addChatMessage("*",  "This is a simple test chat app. ")
        addChatMessage("*", " Welcome to #" + room_name + ". Say hi!");
      }
    } else {
      // A regular chat message.
      if (data.timestamp > lastSeenTimestamp) {
        parseCommand(data.name, data.message);
        lastSeenTimestamp = data.timestamp;
      }
    }
  });

  ws.addEventListener("close", event => {
    console.log("WebSocket closed, reconnecting:", event.code, event.reason);
    rejoin();
  });
  ws.addEventListener("error", event => {
    console.log("WebSocket error, reconnecting:", event);
    rejoin();
  });
}

function addChatMessage(name, text) {
    chatlog = [...chatlog, {name, text}];
    chat_container.scroll({ top: chat_container.scrollHeight, behavior: 'smooth' });
}
onMount(()=> {
    username = localStorage.getItem("username") || "";
})


let chatlog = [];

import {page} from "$app/stores";

let currentWebSocket;
let show_name_form = true;
let show_room_picker = false;
let username;
let room_name = $page.url.hash.substring(1);
let show_chatroom = false;
let message_text;
let chat_container;
let topic;

function parseCommand(user, message) {
    if (message.includes('/topic')) {
        topic = message.replace('/topic', '').trim()
        addChatMessage("*", "Topic set to: " + topic)
    }
    // todo: add start vote command
        // todo: add vote command
        // todo: show results command
        // todo: pick winner command
        // todo: clear voting completely
    else {
        addChatMessage(user, message)
    }
}


$: isAtBottom = chat_container && chat_container.scrollTop + chat_container.clientHeight >= chat_container.scrollHeight


function setUsername(){
    if (username.length < 3) {
      alert("Your name must be at least 3 characters long.");
      return;
    }

    localStorage.setItem("username", username)


    startChat();
    show_name_form = false;
    show_chatroom = true;

}

</script>
<svelte:window on:keydown={handleKeydown} />

<div class="full-screen">

{#if show_name_form}
<form on:submit|preventDefault={setUsername}>
    <label class="label">Enter your name:
      <input class="input" required minlength="3" id="name-input" placeholder="your name" bind:value={username}>
</label>
      <label class="label">Enter or create a room:
      <input class="input" id="room-name" placeholder="room name" bind:value={room_name}>
          </label>
        <button type="button" disabled={!room_name && room_name.length < 3 && username && username.length } class="btn variant-filled" id="go-public" on:click={setUsername} >Go &raquo;</button>

    </form>
{:else if show_chatroom}

    <div class="header">

    <h1>#{room_name}</h1>
    <p>users: {#each users as user}{user}, {/each}</p>
        </div>
    <div bind:this={chat_container} class="chat-container">
    {#each chatlog as {name, text}}
        <p>{name}: {text}</p>

        {/each}
        <div id="anchor"></div>

    </div>


    <div class="bottom-bar my-3">
        {#if topic}
            <div>
            <p>topic: {topic}</p>
                </div>
            {/if}
        <div>
    <form id="chatroom" on:submit|preventDefault={submitMessage}>
      <input class="input" id="chat-input" maxlength="256" bind:value={message_text}>
    </form>
            </div>
        </div>

    {/if}
</div>

<style>
    .input {
        padding-left: 10px;
    }
    .full-screen {
        height: 100vh;
        width: 100vw;
        display: flex;
        flex-direction: column;
        justify-content: center;
        align-items: center;
    }
    .header {
        display: flex;
        flex-direction: column;
        justify-content: center;
        align-items: center;
    }
    .chat-container {
        width: 80vw;
        flex-direction: column;
        overflow-y: scroll;
    }

    .chat-container * {
        overflow-anchor: none;
    }

    #anchor{
        overflow-anchor: auto;
        height:50px;
    }
    #chat-input {
        width: 79vw;
    }
</style>