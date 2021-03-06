<script>
  import { record, mirror } from "rrweb";
  import { createMachine, interpret } from "@xstate/fsm";
  import { onMount, onDestroy } from "svelte";
  import { quintOut } from "svelte/easing";
  import { scale } from "svelte/transition";
  import { RtcTransporter } from "./transport";
  import { SourceBuffer } from "./buffer";
  import {
    EMBED_UID,
    CUSTOM_EVENT_TAGS,
    REMOTE_CONTROL_ACTIONS,
  } from "./constant";
  import { applyMirrorAction } from "./common";
  import Panel from "./components/Panel.svelte";
  import Tag from "./components/Tag.svelte";
  import Icon from './components/Icon.svelte'

  const transporter = new RtcTransporter(EMBED_UID);
  let login = transporter.login();

  let ref;
  $: ref && document.body.appendChild(ref);

  let open = false;

  let stopRecordFn;
  const buffer = new SourceBuffer({
    onTimeout(record) {
      transporter.sendRecord(record);
    },
  });

  const embedMachine = createMachine(
    {
      initial: "idle",
      states: {
        idle: {
          on: {
            START: {
              target: "ready",
              actions: ["start"],
            },
          },
        },
        ready: {
          on: {
            CONNECT: {
              target: "connected",
              actions: ["connect"],
            },
          },
        },
        connected: {
          on: {
            STOP: {
              target: "idle",
              actions: ["stop"],
            },
          },
        },
      },
    },
    {
      actions: {
        start() {
          transporter.sendSourceReady();
        },
        connect() {
          const stopRecord = record({
            emit(event) {
              const id = buffer.add(event);
              transporter.sendRecord(buffer.buffer[id]);
            },
            inlineStylesheet: false
          });
          const timer = setInterval(() => {
            record.addCustomEvent(CUSTOM_EVENT_TAGS.PING);
          }, 1000);
          stopRecordFn = () => {
            stopRecord();
            clearInterval(timer);
          };
        },
        stop() {
          stopRecordFn();
          transporter.sendStop();
          buffer.reset();
        },
      },
    }
  );

  const controlMachine = createMachine(
    {
      initial: "not_control",
      states: {
        not_control: {
          on: {
            REQUEST: {
              target: "requesting",
            },
          },
        },
        requesting: {
          on: {
            ACCEPT: {
              target: "controlled",
              actions: ["accept"],
            },
          },
        },
        controlled: {
          on: {
            STOP: {
              target: "not_control",
            },
          },
        },
      },
    },
    {
      actions: {
        accept() {
          record.addCustomEvent(CUSTOM_EVENT_TAGS.ACCEPT_REMOTE_CONTROL);
        },
      },
    }
  );

  let selecting = false;
  let mask;
  $: mask && document.body.appendChild(mask);
  let blockElSet = new Set();
  $: blockEls = Array.from(blockElSet);

  const highlight = (target) => {
    const { x, y, width, height } = target.getBoundingClientRect();
    Object.assign(mask.style, {
      left: `${x}px`,
      top: `${y}px`,
      width: `${width}px`,
      height: `${height}px`,
      display: 'inherit'
    });
  };
  const removeHighlight = () => {
    Object.assign(mask.style, {
      left: 0,
      top: 0,
      width: 0,
      height: 0,
      display: 'none'
    });
  };

  const over = (event) => {
    if (
      event.target &&
      ref !== event.target &&
      !ref.contains(event.target) &&
      event.target !== document.body
    ) {
      highlight(event.target);
    }
  };
  const click = (event) => {
    if (!selecting || ref.contains(event.target)) {
      return;
    }
    event.target.classList.add("rr-block");
    blockElSet = blockElSet.add(event.target);
    cancelSelect();
  };
  const cancelSelect = () => {
    selecting = false;
    removeHighlight();
    window.removeEventListener("mousemove", over, { capture: true });
    window.removeEventListener("click", click, { capture: true });
  };
  const removeBlockEl = (el) => {
    blockElSet.delete(el);
    blockElSet = blockElSet;
    removeHighlight();
    el.classList.remove("rr-block");
  };
  function handleSelectBlock() {
    if (selecting) {
      cancelSelect();
    } else {
      selecting = true;
      window.addEventListener("mousemove", over, { capture: true });
      window.addEventListener("click", click, { capture: true });
    }
  }

  function changeMouseSize(level) {
    record.addCustomEvent(CUSTOM_EVENT_TAGS.MOUSE_SIZE, { level });
  }

  function stopRemoteControl() {
    record.addCustomEvent(CUSTOM_EVENT_TAGS.STOP_REMOTE_CONTROL);
    controlService.send("STOP");
  }

  let current = embedMachine.initialState;
  const service = interpret(embedMachine);

  let controlCurrent = controlMachine.initialState;
  const controlService = interpret(controlMachine);

  onMount(() => {
    service.start();
    service.subscribe((state) => {
      current = state;
    });
    controlService.start();
    controlService.subscribe((state) => {
      controlCurrent = state;
    });

    transporter.on("mirrorReady", () => {
      service.send("CONNECT");
    });
    transporter.on("ack", ({ payload }) => {
      buffer.delete(payload);
    });
    transporter.on("remoteControl", ({ payload }) => {
      switch (payload.action) {
        case REMOTE_CONTROL_ACTIONS.REQUEST:
          controlService.send("REQUEST");
          break;
        case REMOTE_CONTROL_ACTIONS.STOP:
          controlService.send("STOP");
          break;
        default:
          applyMirrorAction(mirror, payload);
          break;
      }
    });
  });
  onDestroy(() => {
    service.stop();
    controlService.stop();
  });
</script>

<div class="syncit-embed" bind:this="{ref}">
  {#if open}
  <div
    transition:scale="{{duration: 500, opacity: 0.5, easing: quintOut}}"
    style="transform-origin: right bottom;"
  >
    <Panel>
      {#await login}
      <div class="syncit-center syncit-load-text">初始化中...</div>
      {:then}
      <!-- -->
      {#if current.matches('idle')}
      <div class="syncit-center">
        <div class="syncit-panel-control">
          <button class="syncit-btn ordinary" on:click="{handleSelectBlock}">
            {selecting ? '取消' : '选择屏蔽区域'}
          </button>
          <div class="syncit-block-els">
            {#each blockEls as el, idx}
            <Tag
              on:mouseover="{() => highlight(el)}"
              on:mouseout="{removeHighlight}"
              on:click="{() => removeBlockEl(el)}"
              >区域-{idx + 1}
            </Tag>
            {/each}
          </div>
        </div>
        <button class="syncit-btn" on:click="{() => service.send('START')}">
          启用 syncit 分享
        </button>
      </div>
      {:else if current.matches('ready')}
      <div class="syncit-center syncit-load-text">
        <div class="syncit-load-text">
          已启用，等待连接中
        </div>
        <button
          style="margin-top: 8px;"
          class="syncit-btn ordinary"
          on:click="{() => transporter.sendSourceReady()}"
        >
          重试
        </button>
      </div>
      {:else if current.matches('connected')}
      <div class="syncit-center">
        <div class="syncit-panel-control">
          <p>镜像鼠标尺寸</p>
          <div class="syncit-mouses">
            {#each ['小', '中', '大'] as size, idx}
            <button
              class="{`syncit-mouse-${idx + 1}`} syncit-btn ordinary"
              on:click="{() => changeMouseSize(idx + 1)}"
            >
              {size}
            </button>
            {/each}
          </div>
          <p>
            远程控制：
            <!---->
            {#if controlCurrent.matches('not_control')}未启用
            <!---->
            {:else if controlCurrent.matches('requesting')}申请中
            <!---->
            {:else if controlCurrent.matches('controlled')}已启用
            <!---->
            {/if}
          </p>
          {#if controlCurrent.matches('requesting')}
          <button
            class="syncit-btn ordinary"
            on:click="{() => controlService.send('ACCEPT')}"
          >
            允许远程控制
          </button>
          {:else if controlCurrent.matches('controlled')}
          <button class="syncit-btn ordinary" on:click="{stopRemoteControl}">
            终止远程控制
          </button>
          {/if}
        </div>
        <button class="syncit-btn" on:click="{() => service.send('STOP')}">
          停止分享
        </button>
      </div>
      {/if}
      <!---->
      {:catch error}
      <div class="syncit-center syncit-error">{error.message}</div>
      {/await}
    </Panel>
  </div>
  {/if}
  <!---->
  <button class="syncit-toggle syncit-btn" on:click="{() => open = !open}">
    <Icon name={open ? 'close' : 'team'}></Icon>
  </button>
  <!---->
  <div bind:this="{mask}" class="syncit-mask"></div>
</div>

<style>
  button {
    outline: none;
  }

  .syncit-embed {
    position: fixed;
    right: 1em;
    bottom: 1em;
    display: flex;
    flex-direction: column;
    z-index: 9999;
  }

  .syncit-btn:hover {
    background: #3399ff;
  }

  .syncit-btn,
  .syncit-btn:active {
    cursor: pointer;
    background: #0078f0;
    border: 1px solid rgba(62, 70, 82, 0.18);
    box-shadow: 0px 1px 2px rgba(184, 192, 204, 0.6);
    color: #fff;
    padding: 8px 16px;
    border-radius: 4px;
    font-size: 14px;
    line-height: 22px;
    margin-bottom: 0.5em;
  }

  .syncit-btn.ordinary {
    background: #fff;
    color: #3e4652;
    border: 1px solid rgba(129, 138, 153, 0.6);
  }
  .syncit-btn.ordinary:hover {
    background: #f5f7fa;
  }
  .syncit-btn.ordinary:active {
    background: #dfe4eb;
  }

  .syncit-toggle,
  .syncit-toggle:active {
    width: 40px;
    height: 40px;
    line-height: 40px;
    border-radius: 20px;
    padding: 0;
    align-self: flex-end;
  }

  .syncit-center {
    width: 100%;
    height: 100%;
    display: flex;
    align-items: center;
    justify-content: center;
    flex-direction: column;
  }

  .syncit-load-text {
    font-size: 14px;
    line-height: 22px;
    color: #3e4652;
  }

  .syncit-error {
    color: #e75a3a;
  }

  .syncit-mask {
    position: fixed;
    left: 0;
    top: 0;
    pointer-events: none;
    z-index: 999999;
    background: rgba(136, 194, 232, 0.75);
    border: 1px solid #3399ff;
    display: none;
  }

  .syncit-panel-control {
    flex: 1;
    width: 100%;
    align-items: flex-start;
    border-bottom: 1px solid rgba(235, 239, 245, 0.6);
    margin-bottom: 8px;
  }

  .syncit-block-els {
    display: flex;
    overflow: auto;
    word-break: keep-all;
  }

  .syncit-block-els > :global(span) {
    cursor: pointer;
    margin-right: 4px;
  }

  .syncit-mouses {
    display: flex;
    justify-content: space-between;
    align-items: center;
  }
</style>
