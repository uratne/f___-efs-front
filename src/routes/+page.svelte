<!-- src/routes/+page.svelte -->
<script lang="ts">
    import { onMount, onDestroy, tick } from 'svelte';
    import { fade } from 'svelte/transition';
  
    let lines: Array<{ content: string; timestamp: string; type: 'data' | 'system' }> = [];
    let autoScroll = true;
    let terminalDiv: HTMLDivElement;
    let eventSource: EventSource | null = null;
  
    async function scrollToBottom() {
        if (autoScroll && terminalDiv) {
            // Wait for DOM update
            await tick();
            terminalDiv.scrollTo({
                top: terminalDiv.scrollHeight,
                behavior: 'smooth'
            });
        }
    }

    async function handleMessage(event: MessageEvent) {
        const message = JSON.parse(event.data);

        if (message.type === 'Data') {
            if (message.replace_last_row && lines.length > 0) {
                lines = [...lines.slice(0, -1), {
                    content: message.row,
                    timestamp: message.timestamp,
                    type: 'data'
                }];
            } else {
                lines = [...lines, {
                    content: message.row,
                    timestamp: message.timestamp,
                    type: 'data'
                }];
            }
        } else if (message.type === 'System') {
            lines = [...lines, {
                content: message.message,
                timestamp: message.timestamp,
                type: 'system'
            }];
        }
;
        await scrollToBottom();
    }
  
    function handleScroll() {
        if (!terminalDiv) return;
        const isAtBottom = Math.abs(terminalDiv.scrollHeight - terminalDiv.clientHeight - terminalDiv.scrollTop) < 10;
        autoScroll = isAtBottom;
    }
  
    onMount(() => {
        const application = { SinglePod: "Fefs 1.0" };
        const encodedApp = encodeURIComponent(JSON.stringify(application));
        eventSource = new EventSource(`/api/sse?application=${encodedApp}`);
        
        eventSource.onmessage = handleMessage;
        eventSource.onerror = async (error) => {
            console.error('SSE Error:', error);
            lines = [...lines, {
                content: 'Connection error',
                timestamp: new Date().toISOString(),
                type: 'system'
            }];
            await scrollToBottom();
        };
    });
  
    onDestroy(() => {
        if (eventSource) {
            eventSource.close();
        }
    });
</script>

<div class="terminal">
    <div class="controls">
        <label class="auto-scroll">
            <input type="checkbox" bind:checked={autoScroll} />
            Auto-scroll
        </label>
        <button on:click={() => lines = []}>Clear Terminal</button>
    </div>
  
    <div class="content" bind:this={terminalDiv} on:scroll={handleScroll}>
        {#each lines as line (line.timestamp + line.content)}
            <div class="line" class:system={line.type === 'system'} transition:fade={{ duration: 150 }}>
                <span class="timestamp">{new Date(line.timestamp + 'z').toLocaleTimeString()}</span>
                <span class="line-content">{line.content}</span>
            </div>
        {/each}
    </div>
</div>
  
<style>
    /* Reset all defaults */
    :global(html), :global(body) {
        margin: 0;
        padding: 0;
        width: 100%;
        height: 100%;
        overflow: hidden;
    }

    :global(*) {
        box-sizing: border-box;
    }

    :global(#svelte) {
        width: 100%;
        height: 100%;
        position: absolute;
        top: 0;
        left: 0;
    }

    .terminal {
        position: fixed;
        top: 0;
        left: 0;
        right: 0;
        bottom: 0;
        background-color: #000;
        color: #fff;
        font-family: 'Consolas', 'Monaco', monospace;
        font-size: 14px;
        line-height: 1.4;
        display: flex;
        flex-direction: column;
    }

    .controls {
        background-color: #1a1a1a;
        padding: 8px;
        display: flex;
        gap: 1rem;
        border-bottom: 1px solid #333;
        width: 100%;
    }

    .auto-scroll {
        display: flex;
        align-items: center;
        gap: 0.5rem;
        color: #888;
    }

    button {
        padding: 4px 12px;
        background-color: #333;
        border: 1px solid #444;
        color: #888;
        border-radius: 2px;
        cursor: pointer;
    }

    button:hover {
        background-color: #444;
    }

    .content {
        flex: 1;
        overflow-y: auto;
        padding: 8px;
        width: 100%;
    }

    .line {
        display: flex;
        gap: 1rem;
        padding: 2px 0;
        width: 100%;
    }

    .line.system {
        color: #ffd700;
    }

    .timestamp {
        color: #666;
        min-width: 100px;
        flex-shrink: 0;
    }

    .line-content {
        flex: 1;
        white-space: pre-wrap;
        word-break: break-all;
    }

    /* Scrollbar styling */
    .content::-webkit-scrollbar {
        width: 10px;
    }

    .content::-webkit-scrollbar-track {
        background: #0a0a0a;
    }

    .content::-webkit-scrollbar-thumb {
        background-color: #333;
        border-radius: 5px;
        border: 2px solid #0a0a0a;
    }

    /* Input styles */
    input[type="checkbox"] {
        accent-color: #444;
    }
</style>