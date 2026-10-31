<script lang="ts">
    import { onMount } from "svelte";
    import { writeText, readText } from "@tauri-apps/plugin-clipboard-manager";
    import { Copy, ClipboardPaste } from "@lucide/svelte";

    let colorField: HTMLButtonElement;
    let colorSelector: HTMLDivElement;
    let hue: number = 360;
    let alpha: number = 1;
    let saturation: number = 0;
    let brightness: number = 1;
    let color: string = "#ffffff";
    let updatingColorSelector: boolean = false;

    function tryGetHex(value: string): string | undefined {
        value = value.trim().toLowerCase();

        if (!value.startsWith("#")) {
            value = "#" + value;
        }

        if (/^#[0-9a-fA-F]{6}$/.test(value)) {
            return value;
        }
    }

    function hexToHsv(hex: string) {
        const r = parseInt(hex.slice(1, 3), 16) / 255;
        const g = parseInt(hex.slice(3, 5), 16) / 255;
        const b = parseInt(hex.slice(5, 7), 16) / 255;

        const max = Math.max(r, g, b);
        const min = Math.min(r, g, b);
        const delta = max - min;

        let h = 0;
        const s = max === 0 ? 0 : delta / max;
        const v = max;

        if (delta !== 0) {
            if (max === r) h = 60 * (((g - b) / delta) % 6);
            else if (max === g) h = 60 * ((b - r) / delta + 2);
            else h = 60 * ((r - g) / delta + 4);
        }

        return { h: (h + 360) % 360, s, v };
    }

    function hsvToHex(h: number, s: number, v: number) {
        const f = (n: number, k = (n + h / 60) % 6) =>
            v - v * s * Math.max(0, Math.min(k, 4 - k, 1));

        const toHex = (n: number) =>
            Math.round(n * 255)
                .toString(16)
                .padStart(2, "0");

        return `#${toHex(f(5))}${toHex(f(3))}${toHex(f(1))}`;
    }

    function setColor(h: number, s: number, v: number) {
        hue = h;

        const rect = colorField.getBoundingClientRect();
        const radius = colorSelector.offsetWidth / 2;
        const x = radius + s * (rect.width - radius * 2);
        const y = radius + (1 - v) * (rect.height - radius * 2);

        colorSelector.style.left = x + "px";
        colorSelector.style.top = y + "px";
    }

    function updateColorSelector(event: MouseEvent) {
        if (event.buttons !== 1 || !updatingColorSelector) {
            return;
        }

        const rect = colorField.getBoundingClientRect();
        const radius = colorSelector.offsetWidth / 2;
        const x = Math.max(
            radius,
            Math.min(event.clientX - rect.left, rect.width - radius),
        );
        const y = Math.max(
            radius,
            Math.min(event.clientY - rect.top, rect.height - radius),
        );

        colorSelector.style.left = x + "px";
        colorSelector.style.top = y + "px";

        saturation = (x - radius) / (rect.width - radius * 2);
        brightness = 1 - (y - radius) / (rect.height - radius * 2);
        color = hsvToHex(hue, saturation, brightness);
    }

    onMount(() => {
        setColor(360, 0, 1);
    });
</script>

<svelte:window
    onmousemove={updateColorSelector}
    onmouseup={() => (updatingColorSelector = false)}
/>

<div class="app">
    <button
        class="color-field"
        style={`--hue: ${hue}; --alpha: ${alpha}`}
        onmousedown={() => (updatingColorSelector = true)}
        onmouseup={() => (updatingColorSelector = false)}
        title=""
        bind:this={colorField}
    >
        <div class="color-selector" bind:this={colorSelector}></div>
    </button>

    <input
        type="range"
        min="0"
        max="360"
        bind:value={hue}
        oninput={() => (color = hsvToHex(hue, saturation, brightness))}
    />

    <!-- <input type="range" max="1" step="0.1" bind:value={alpha} /> -->

    <div class="actions">
        <button
            title=""
            onclick={async () => {
                const cb = await readText();
                const hex = tryGetHex(cb);
                if (!hex) return;

                color = hex;
                const { h, s, v } = hexToHsv(hex);
                setColor(h, s, v);
            }}
        >
            <ClipboardPaste color="var(--fg-primary)" />
        </button>

        <input
            type="text"
            class="color-text"
            bind:value={color}
            oninput={() => {
                const hex = tryGetHex(color);
                if (!hex) return;

                color = hex;
                const { h, s, v } = hexToHsv(hex);
                setColor(h, s, v);
            }}
            onblur={() => {
                const hex = tryGetHex(color);
                if (!hex) color = hsvToHex(hue, saturation, brightness);
            }}
        />

        <button title="" onclick={async () => await writeText(color)}>
            <Copy color="var(--fg-primary)" />
        </button>
    </div>
</div>

<style>
    :global(*) {
        box-sizing: border-box;
        margin: 0;
    }

    :global(html),
    :global(body) {
        width: 100%;
        height: 100%;
        overflow: hidden;
    }

    :root {
        --fg-primary: #bababa;
        --bg-primary: #1a1a1a;
        --border-primary: #383838;
        --btn: #242424;
        --btn-hover: #2e2e2e;
        --btn-active: #1f1f1f;
        --rounding: 3px;
        --padding: 12px;
    }

    .app {
        width: 100%;
        height: 100%;
        background-color: var(--bg-primary);
        border: 2px solid var(--border-primary);
        padding: var(--padding);
        display: flex;
        flex-direction: column;
        gap: 6px;
        display: flex;
        flex-direction: column;
        justify-content: center;
        /*align-items: center;*/
    }

    .color-field {
        position: relative;
        width: 100%;
        height: 250px;
        border: none;
        border-radius: var(--rounding);
        outline: none;
        background:
            linear-gradient(to top, black, transparent),
            linear-gradient(
                to right,
                white,
                hsla(var(--hue), 100%, 50%, var(--alpha))
            );
    }

    input {
        width: 100%;
        height: 30px;
    }

    .color-selector {
        position: absolute;
        width: 15px;
        height: 15px;
        border: 3px solid black;
        border-radius: var(--rounding);
        outline: 2px solid white;
        transform: translate(-50%, -50%);
    }

    .actions {
        display: flex;
        justify-content: center;
        align-items: center;
        gap: 6px;
    }

    input[type="range"]::-webkit-slider-runnable-track {
        height: 24px;
        background-color: black;
        border-radius: var(--rounding);
        background: linear-gradient(
            to right,
            red,
            yellow,
            lime,
            cyan,
            blue,
            magenta,
            red
        );
    }

    input[type="range"]::-webkit-slider-thumb {
        -webkit-appearance: none !important;
        appearance: none !important;
        height: 24px;
        width: 24px;
    }

    .color-text {
        height: 50px;
        text-align: center;
        font-size: 24px;
        color: var(--fg-primary);
        background-color: var(--btn);
        border-radius: var(--rounding);
        border: none;
        outline: none;
    }

    .color-text:focus {
        border: 3px solid var(--border-primary);
    }

    button {
        width: 50px;
        height: 50px;
        background-color: var(--btn);
        border-radius: var(--rounding);
        border: none;
        outline: none;
        cursor: pointer;
        display: flex;
        justify-content: center;
        align-items: center;
    }

    button:hover {
        background-color: var(--btn-hover);
    }

    button:active {
        background-color: var(--btn-active);
    }
</style>
