<script lang="ts">
    import {Delta, Editor, EditorChangeEvent, EditorRange, placeholder, TextChange,} from "typewriter-editor";
    import {onMount, SvelteComponentTyped} from "svelte";
    import {fade} from 'svelte/transition';
    import {BehaviorSubject, fromEvent, merge, Observable, Subject} from "rxjs";
    import {filter, skip, take, takeWhile} from "rxjs/operators";
    import {UndoStack} from "typewriter-editor/lib/modules/history";
    import DragHandle from './DragHandle.svelte';
    import InlineMenu from 'typewriter-editor/lib/InlineMenu.svelte';

    window['Delta'] = Delta;
    let container: HTMLDivElement;
    let cachedHistory: UndoStack;
    let editor: Editor;
    let inlineMenu;
    let localEditor = new BehaviorSubject<Editor>(null);
    const dragHandle: SvelteComponentTyped = new DragHandle({target: document.body, props: {element: null}});
    const destroyLocalEditor = new Subject();

    onMount(() => {
        editor = window['editor'] = new Editor({
            root: container,
        });

        editor.setDelta(new Delta({
            ops: [
                {
                    insert: "Heading 1"
                },
                {
                    insert: "\n",
                    attributes: {
                        header: 1
                    }
                },
                {
                    insert: "Short text"
                },
                {
                    insert: "\n",
                },
                {
                    insert: "Heading 2",
                },
                {
                    insert: "\n",
                    attributes: {
                        header: 2
                    }
                },
                {
                    insert: "Short text 2"
                },
                {
                    insert: "\n"
                }
            ]
        }))
    });

    function isRedoShortcut(event) {
        return (event.ctrlKey || event.metaKey) && event.key == 'y';
    }

    function isUndoShortcut(event) {
        return (event.ctrlKey || event.metaKey) && event.key == 'z'
    }

    async function handleShortcuts(event: KeyboardEvent) {
        if ($localEditor) {
            if (event.key === 'Enter' && !event.shiftKey) {
                event.preventDefault();
                event.stopPropagation();

                const line = editor.doc.getLineAt(editor.doc.selection[0]);
                destroyLocalEditor.next();

                await new Promise(requestAnimationFrame);
                const endOfLine = editor.doc.getLineRange(line.attributes.id)[1];
                editor.update(new Delta().retain(endOfLine).insert('').insert('\n'));

                setTimeout(() => {
                    editor.select(endOfLine);
                    handleClick();
                })
            }
            if (event.key === 'Escape') {
                destroyLocalEditor.next();
            }
            if (event.key === 'ArrowDown' && event.altKey) {
                // Navigate to next block
                event.preventDefault();
                event.stopPropagation()

                const currentBlock = editor.doc.getLineAt(editor.doc.selection[1]);
                const nextBlock = editor.doc.lines[editor.doc.lines.indexOf(currentBlock) + 1];
                destroyLocalEditor.next();

                await new Promise(requestAnimationFrame);
                editor.select(editor.doc.getLineRange(nextBlock));
                onSelectionChange();
            }
            if (event.key === 'ArrowUp' && event.altKey) {
                // Navigate to previous block
                event.preventDefault();
                event.stopPropagation()

                const currentBlock = editor.doc.getLineAt(editor.doc.selection[1]);
                const previousBlock = editor.doc.lines[editor.doc.lines.indexOf(currentBlock) - 1];
                destroyLocalEditor.next();

                await new Promise(requestAnimationFrame);
                editor.select(editor.doc.getLineRange(previousBlock));
                onSelectionChange();
            }
        } else {
            if (!event.ctrlKey && !event.metaKey) {
                event.stopPropagation();
                event.preventDefault()
            }

            if (isUndoShortcut(event) && editor.modules.history.hasUndo()) {
                editor.modules.history.undo();
            }

            if (isRedoShortcut(event) && editor.modules.history.hasRedo()) {
                editor.modules.history.redo();
            }

            await new Promise(requestAnimationFrame);
            if (editor.doc.selection) {
                editor.select(0);
            }
        }
    }

    function enableEditor() {
        editor.setRoot(container);
        if (cachedHistory) {
            editor.modules.history.setStack(cachedHistory);
        }
        editor.enabled = true;
        container.setAttribute('contentEditable', 'false');
    }

    function disableEditor() {
        saveHistory();
        editor.enabled = false;
        const temp = document.createElement('div');
        editor.setRoot(temp);
    }

    function injectLocalEditor(lineRange: EditorRange, cursorPos: number, element: HTMLElement): { localEditorContainer: HTMLDivElement, localChanges: Observable<EditorChangeEvent> } {
        const localEditorContainer = document.createElement('div');
        localEditorContainer.classList.add('inline-edit');
        element.insertAdjacentElement('beforebegin', localEditorContainer);
        element.replaceWith(localEditorContainer)

        const _localEditor = new Editor({root: localEditorContainer, html: element.outerHTML});
        localEditor.next(_localEditor);
        localEditorContainer.firstElementChild.classList.add('editing');
        _localEditor.select(cursorPos);
        const localChanges: Observable<EditorChangeEvent> = fromEvent(_localEditor, 'change');

        return {localEditorContainer, localChanges};
    }

    function handleClick(event?: MouseEvent) {
        const target: HTMLElement = (event?.target as HTMLElement);
        if (target?.tagName === 'HR') {
            return;
        }

        if ($localEditor) return;
        onSelectionChange();
    }

    function onSelectionChange() {
        const selection = document.getSelection();
        const element = selection.anchorNode.nodeName === '#text' ? selection.anchorNode.parentElement : selection.anchorNode as HTMLElement;
        const lineId = element['key'];
        const lineRange = editor.doc.getLineRange(lineId);
        const changes = [];

        if (!lineRange) return;

        enableEditor();

        const {
            localEditorContainer,
            localChanges
        } = injectLocalEditor(lineRange, selection.focusOffset, element);

        new Promise(requestAnimationFrame).then(() => {
            disableEditor();
            localEditorContainer.focus()
        })

        const changeSubscription = localChanges.subscribe(changeEvent => {
            localEditorContainer.firstElementChild.classList.add('editing');
            const change = changeEvent.change.delta;
            if (change.ops.length == 0) return;
            changes.push(change)
        });

        merge(
            destroyLocalEditor,
            fromEvent(document, 'click')
                .pipe(
                    skip(1),
                    filter(click => !click.composedPath().includes(localEditorContainer))
                )
        ).pipe(take(1)).subscribe(_ => {
            changeSubscription.unsubscribe();
            const _localEditor: Editor = $localEditor as Editor;
            const history = _localEditor.modules.history.getStack();
            _localEditor.destroy();
            localEditor.next(null);
            const composed = new Delta(changes.reduce((acc, curr) => {
                return acc.compose(curr)
            }, new Delta()));
            enableEditor();
            editor.update(new Delta().retain(lineRange?.[0]).concat(composed));
            editor.select(0);
            mergeHistory(history, lineRange);
        });
    }

    function retrieveHistory() {
        return cachedHistory;
    }

    function saveHistory() {
        cachedHistory = editor.modules.history.getStack();
    }

    function mergeHistory(historyStack: UndoStack, [changeStartIndex, _]) {
        if (historyStack.undo.length === 0 && historyStack.redo.length === 0) return;

        const currentStack: UndoStack = retrieveHistory();
        const undo = historyStack.undo.map(it => {
            let redo: TextChange = it.redo;
            let undo: TextChange = it.undo;

            Object.keys((undo.delta.ops[0])[0] !== 'retain' || undo.delta.ops[0].retain !== changeStartIndex) && changeStartIndex !== 0 ? undo.delta.ops.splice(0, 0, {retain: changeStartIndex}) : null;
            Object.keys((redo.delta.ops[0])[0] !== 'retain' || redo.delta.ops[0].retain !== changeStartIndex) && changeStartIndex !== 0 ? redo.delta.ops.splice(0, 0, {retain: changeStartIndex}) : null;

            return {
                redo,
                undo
            }
        });

        currentStack.undo.pop();
        currentStack.undo = [...currentStack.undo, ...undo];
        currentStack.redo = [];
        editor.modules.history.setStack(currentStack);
    }

    function showDragHandle(event: MouseEvent) {
        if (dragHandle.dragging === true) return;
        const target = event.target as HTMLElement;

        if (target.tagName === 'DIV') return;

        dragHandle.$set({element: target});

        const handleElement = dragHandle.$$.ctx[0];

        fromEvent(document, 'mousemove')
            .pipe(
                filter(evt => ![target, handleElement, ...handleElement.children].includes(evt.target)),
                takeWhile(() => dragHandle.dragging == false),
                take(1)
            ).subscribe(_ => {
            dragHandle.$set({element: null})
        });
    }

    function reorderBlock(event) {
        const {targetKey, sourceKey} = event.detail;

        const sourceBlock = editor.doc.byId[sourceKey];
        const newLocation = editor.doc.getLineRange(targetKey)?.[0];
        const oldLocation = editor.doc.getLineRange(sourceBlock)?.[0];

        if (newLocation == null) return;

        if (newLocation < oldLocation) {
            // Insert first THEN delete
            const delta = new Delta().retain(newLocation).concat(sourceBlock.content).insert('\n', sourceBlock.attributes).compose(new Delta().retain(oldLocation + sourceBlock.length).delete(sourceBlock.length));
            editor.update(delta);
        } else {
            // Delete first THEN insert
            const delta = new Delta().retain(oldLocation).delete(sourceBlock.length).compose(new Delta().retain(newLocation - sourceBlock.length).concat(sourceBlock.content).insert('\n', sourceBlock.attributes));
            editor.update(delta);
        }
    }
</script>

<style lang="scss">
  * {
    word-break: break-word;
  }

  :focus {
    outline: none;
  }

  .container {
    overflow: hidden;
    padding: 1.5rem;
  }

  .container :global(div.inline-edit:focus) {
    outline: #70708a auto;
  }

  .container :global(:not(div)) {
    position: relative;

    &.drop-target {
      &:after {
        border: 1px solid #70708a;
        content: "";
        display: flex;
        position: absolute;
        top: -8px;
        width: 100%;
      }
    }

    &:not(.drop-target, .editing):hover {
      &:not(hr) {
        box-shadow: -2px 0 0 4px #f5f5f5;
        border-radius: 1px;
        background: #f5f5f5;
      }

      // Create a 'bridge' from the element to the drag handle so it doesn't lose it's hover state
      &:before {
        content: "";
        height: 24px;
        left: -13.3px;
        padding: 0 0 0 2rem;
        position: absolute;
        top: 0px;
        width: 0px;
      }
    }
  }

  :global(hr) {
    overflow: visible;
    box-sizing: border-box;
    margin: 0;
    padding: 14px 0;
    border: none;
    cursor: default;
    height: 1px;
    background-color: #ccc;
    background-clip: content-box;

    &.selected {
      box-shadow: inset 0 0 0 16px #8abbff33;
      outline: 1px solid #70708a;
      outline-offset: -1px;
    }
  }

  :global(.inline-menu) {
    z-index: 999;
  }

  .menu {
    display: flex;
    height: 32px;
    color: #999;
    white-space: nowrap;
  }

  .menu-button {
    text-align: center;
    border: none;
    margin: 0;
    padding: 0;
    width: 48px;
    height: 32px;
    line-height: 32px;
    color: inherit;
    font-size: 12px;
    background: none;
    outline: none;
    cursor: pointer;
  }

  .menu-button:hover {
    color: #444;
  }

  .separator {
    height: 16px;
    margin: 8px 0;
    border-right: 1px solid #aaa;
  }
</style>

<svelte:window on:block-dropped={reorderBlock} on:keydown|capture={handleShortcuts}/>
<section class="container">
        <InlineMenu bind:this={inlineMenu} editor={$localEditor} let:active let:commands>
            <div class="menu" in:fade={{ duration: 100 }}>
                <button
                        class="menu-button"
                        class:active={active.header === 1}
                        on:click={commands.header1}
                >H1
                </button>
                <span class="separator"></span>
                <button
                        class="menu-button"
                        class:active={active.header === 2}
                        on:click={commands.header2}
                >H2
                </button>
                <span class="separator"></span>
                <button
                        class="menu-button"
                        class:active={active.header === 3}
                        on:click={commands.header3}
                >H3
                </button>
                <span class="separator"></span>
                <button
                        class="menu-button"
                        class:active={active.header === 4}
                        on:click={commands.header4}
                >H4
                </button>
                <span class="separator"></span>
                <button
                        class="menu-button"
                        class:active={active.hr}
                        on:click={commands.hr}
                >–
                </button>
            </div>
        </InlineMenu>
    <div bind:this={container} class="content" on:beforeInput={null} on:click|preventDefault={handleClick}
         on:mouseover|preventDefault={showDragHandle}>
    </div>
</section>
