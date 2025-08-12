<script lang="ts">
    export let checked: boolean = false;
    export let onChange: (event: Event & {
        currentTarget: EventTarget & HTMLInputElement;
    }) => void = () => {};
</script>

<label class="switch">
    <input
        type="checkbox"
        bind:checked={checked}
        on:change={(event) => { onChange(event) }}
    />
    <span class="slider"></span>
</label>

<style>
    /* The switch - the box around the slider */
    .switch {
        font-size: 13px;
        position: relative;
        display: inline-block;
        width: 3.5em;
        height: 2em;
    }

    /* Hide default HTML checkbox */
    .switch input {
        opacity: 0;
        width: 0;
        height: 0;
    }

    /* The slider */
    .slider {
        position: absolute;
        cursor: pointer;
        inset: 0;
        background: #244155;
        border-radius: 50px;
        transition: all 0.4s cubic-bezier(0.175, 0.885, 0.32, 1.275);
    }

    .slider:before {
        position: absolute;
        content: "";
        display: flex;
        align-items: center;
        justify-content: center;
        height: 2em;
        width: 2em;
        inset: 0;
        background-color: white;
        border-radius: 50px;
        box-shadow: 0 10px 20px rgba(0, 0, 0, 0.4);
        transition: all 0.4s cubic-bezier(0.175, 0.885, 0.32, 1.275);
    }

    .switch input:checked + .slider {
        background: #826ae3;
    }

    .switch input:checked + .slider:before {
        transform: translateX(1.6em);
    }
</style>