<script lang="ts">
    import type { Show } from "../../../types/Show"
    import { getQuickExample } from "../../converters/txt"
    import { includeEmptySlides, os, textEditActive, textEditZoom } from "../../stores"
    import { transposeText } from "../../utils/chordTranspose"
    import Icon from "../helpers/Icon.svelte"
    import FloatingInputs from "../input/FloatingInputs.svelte"
    import MaterialButton from "../inputs/MaterialButton.svelte"
    import MaterialZoom from "../inputs/MaterialZoom.svelte"
    import { formatText } from "./formatTextEditor"
    import { getPlainEditorText } from "./getTextEditor"
    import Notes from "./tools/Notes.svelte"

    export let currentShow: Show | undefined

    $: allowEmpty = $includeEmptySlides

    let text = ""
    $: if (currentShow) text = getPlainEditorText("", allowEmpty)

    // transpose chords
    function transposeUp() {
        formatText(transposeText(text, 1))
    }
    function transposeDown() {
        formatText(transposeText(text, -1))
    }

    // keyboard shortcuts
    function keydown(e: KeyboardEvent) {
        const target = e.target as HTMLTextAreaElement
        if (!target || target.tagName !== "TEXTAREA") return

        const isMac = $os.platform === 'darwin'
        const cmdOrCtrl = isMac ? e.metaKey : e.ctrlKey
        const altOrOption = e.altKey

        // Line-based cut/copy/paste operations
        if (cmdOrCtrl && (e.key === 'x' || e.key === 'c' || e.key === 'v')) {
            const start = target.selectionStart
            const end = target.selectionEnd
            
            // If no selection, operate on current line
            if (start === end) {
                e.preventDefault()
                
                const lines = target.value.split('\n')
                const cursorPos = start
                let lineStart = 0
                let currentLineIndex = 0
                
                // Find which line the cursor is on
                for (let i = 0; i < lines.length; i++) {
                    const lineEnd = lineStart + lines[i].length
                    if (cursorPos >= lineStart && cursorPos <= lineEnd) {
                        currentLineIndex = i
                        break
                    }
                    lineStart = lineEnd + 1
                }
                
                const currentLine = lines[currentLineIndex]
                
                if (e.key === 'x') {
                    // Cut entire line
                    navigator.clipboard.writeText(currentLine + '\n')
                    lines.splice(currentLineIndex, 1)
                    text = lines.join('\n')
                    formatText(text)
                } else if (e.key === 'c') {
                    // Copy entire line
                    navigator.clipboard.writeText(currentLine + '\n')
                } else if (e.key === 'v') {
                    // Paste at current line
                    navigator.clipboard.readText().then(clipboardText => {
                        lines.splice(currentLineIndex + 1, 0, clipboardText.replace(/\n$/, ''))
                        text = lines.join('\n')
                        formatText(text)
                    })
                }
            }
        }
        
        // Move line up/down with Alt+Arrow
        if (altOrOption && (e.key === 'ArrowUp' || e.key === 'ArrowDown')) {
            e.preventDefault()
            
            const start = target.selectionStart
            const lines = target.value.split('\n')
            let lineStart = 0
            let currentLineIndex = 0
            
            // Find which line the cursor is on
            for (let i = 0; i < lines.length; i++) {
                const lineEnd = lineStart + lines[i].length
                if (start >= lineStart && start <= lineEnd) {
                    currentLineIndex = i
                    break
                }
                lineStart = lineEnd + 1
            }
            
            const direction = e.key === 'ArrowUp' ? -1 : 1
            const newIndex = currentLineIndex + direction
            
            // Check bounds
            if (newIndex >= 0 && newIndex < lines.length) {
                // Swap lines
                const currentLine = lines[currentLineIndex]
                lines[currentLineIndex] = lines[newIndex]
                lines[newIndex] = currentLine
                
                text = lines.join('\n')
                formatText(text)
                
                // Update cursor position to follow the moved line
                setTimeout(() => {
                    const newLineStart = lines.slice(0, newIndex).join('\n').length + (newIndex > 0 ? 1 : 0)
                    const cursorOffset = start - lineStart
                    target.setSelectionRange(newLineStart + cursorOffset, newLineStart + cursorOffset)
                })
            }
        }
    }

    $: showHasChords = Object.values(currentShow?.slides || {}).find((a) => a.items?.find((a) => a.lines?.find((a) => a.chords)))
</script>

<svelte:window on:keydown={keydown} />

<Notes disabled={currentShow?.locked} style="padding: 30px;height: calc(100% - 28px);font-size: {$textEditZoom / 8}em;" placeholder={getQuickExample()} value={text} on:change={(e) => formatText(e.detail)} />

<FloatingInputs arrow let:open>
    <MaterialZoom hidden={!open} columns={$textEditZoom / 10} min={0.5} max={2} defaultValue={1} addValue={-0.1} on:change={(e) => textEditZoom.set(e.detail * 10)} />

    {#if open}
        <div class="divider"></div>
    {/if}

    <MaterialButton isActive title="show.text" on:click={() => textEditActive.set(false)}>
        <Icon id="text_edit" white />
    </MaterialButton>
</FloatingInputs>

{#if showHasChords}
    <FloatingInputs side="left">
        <MaterialButton on:click={transposeUp} title="edit.transpose_up">
            <Icon id="arrow_up" size={1.3} white />
        </MaterialButton>
        <MaterialButton on:click={transposeDown} title="edit.transpose_down">
            <Icon id="arrow_down" size={1.3} white />
        </MaterialButton>
    </FloatingInputs>
{/if}
