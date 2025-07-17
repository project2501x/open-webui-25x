<script lang="ts">
  import { onMount, onDestroy, createEventDispatcher } from 'svelte';
  import { Editor } from '@tiptap/core';
  import StarterKit from '@tiptap/starter-kit';
  import { TextSelection } from 'prosemirror-state';

  export let content = '';
  export let placeholder = 'Type your message...';
  export let autofocus = false;
  export let onKeydown: Function = () => {};

  let element: HTMLDivElement;
  let editor: Editor;
  const dispatch = createEventDispatcher();

  // Variable replacement functionality
  export function replaceVariables(variables: Record<string, any>) {
    if (!editor) return;
    
    const { state, view } = editor;
    const { doc } = state;
    let tr = state.tr;
    const replacements = [];

    doc.descendants((node, pos) => {
      if (node.isText && node.text) {
        const text = node.text;
        const replacedText = text.replace(
          /{{\s*([^|}]+)(?:\|[^}]*)?\s*}}/g,
          (match, varName) => {
            const trimmedVarName = varName.trim();
            return variables.hasOwnProperty(trimmedVarName)
              ? String(variables[trimmedVarName])
              : match;
          }
        );

        if (replacedText !== text) {
          replacements.push({
            from: pos,
            to: pos + text.length,
            text: replacedText
          });
        }
      }
    });

    // Apply replacements in reverse order to maintain correct positions
    replacements.reverse().forEach(({ from, to, text }) => {
      tr = tr.replaceWith(from, to, text !== '' ? state.schema.text(text) : []);
    });

    if (replacements.length > 0) {
      view.dispatch(tr);
    }
  }

  // Inline variable input functionality
  export function insertInlineVariableInput(variableName: string, defaultValue: string = '') {
    if (!editor) return;
    
    // Create an inline input element
    const inputElement = document.createElement('input');
    inputElement.type = 'text';
    inputElement.value = defaultValue;
    inputElement.placeholder = `Enter ${variableName}`;
    inputElement.className = 'inline-variable-input';
    inputElement.style.cssText = `
      display: inline-block;
      border: 1px solid #3b82f6;
      border-radius: 4px;
      padding: 2px 6px;
      margin: 0 2px;
      font-size: inherit;
      background: #eff6ff;
      color: #1e40af;
      min-width: 80px;
      outline: none;
    `;
    
    // Handle input changes
    inputElement.addEventListener('input', (e) => {
      const value = (e.target as HTMLInputElement).value;
      // Update the variable value in the editor
      updateVariableValue(variableName, value);
    });
    
    // Handle focus/blur
    inputElement.addEventListener('focus', () => {
      inputElement.style.borderColor = '#1d4ed8';
      inputElement.style.background = '#dbeafe';
    });
    
    inputElement.addEventListener('blur', () => {
      inputElement.style.borderColor = '#3b82f6';
      inputElement.style.background = '#eff6ff';
    });
    
    // Insert the input element at cursor position
    const { state, view } = editor;
    const { selection } = state;
    const { from, to } = selection;
    
    // Replace the {{VARIABLE}} text with the input element
    const tr = state.tr.replaceWith(from, to, state.schema.text(''));
    view.dispatch(tr);
    
    // Insert the input element
    const range = document.createRange();
    const selection2 = window.getSelection();
    if (selection2 && selection2.rangeCount > 0) {
      range.setStart(selection2.getRangeAt(0).startContainer, selection2.getRangeAt(0).startOffset);
      range.insertNode(inputElement);
      inputElement.focus();
    }
  }

  // Update variable value in the editor
  function updateVariableValue(variableName: string, value: string) {
    if (!editor) return;
    
    const { state, view } = editor;
    const { doc } = state;
    let tr = state.tr;
    
    doc.descendants((node, pos) => {
      if (node.isText && node.text) {
        const text = node.text;
        const updatedText = text.replace(
          new RegExp(`{{\\s*${variableName}\\s*}}`, 'g'),
          value
        );
        
        if (updatedText !== text) {
          tr = tr.replaceWith(pos, pos + text.length, state.schema.text(updatedText));
        }
      }
    });
    
    view.dispatch(tr);
  }

  // Command replacement
  export function replaceCommandWithText(text: string) {
    if (!editor) return;
    
    const { state, view } = editor;
    const { selection } = state;
    const pos = selection.from;
    
    // Find word boundaries at cursor
    const { start, end } = getWordBoundsAtPos(state.doc, pos);
    
    const tr = state.tr.replaceWith(
      start,
      end,
      text !== '' ? state.schema.text(text) : []
    );
    
    view.dispatch(tr);
  }

  // Get word at cursor position
  export function getWordAtDocPos(): string {
    if (!editor) return '';
    
    const { state } = editor;
    const { selection } = state;
    const pos = selection.from;
    
    const { start, end } = getWordBoundsAtPos(state.doc, pos);
    return state.doc.textBetween(start, end);
  }

  // Helper function to get word boundaries
  function getWordBoundsAtPos(doc: any, pos: number) {
    let start = pos;
    let end = pos;
    
    // Find start of word
    while (start > 0 && !/\s/.test(doc.textBetween(start - 1, start))) {
      start--;
    }
    
    // Find end of word
    while (end < doc.content.size && !/\s/.test(doc.textBetween(end, end + 1))) {
      end++;
    }
    
    return { start, end };
  }

  // Insert content at cursor
  export function insertContent(content: string) {
    if (!editor) return;
    editor.commands.insertContent(content);
  }

  // Set text content
  export function setText(text: string) {
    if (!editor) return;
    editor.commands.setContent(text);
  }

  // Focus the editor
  export function focus() {
    if (editor) {
      editor.commands.focus();
    }
  }

  // Clear content
  export function clear() {
    if (editor) {
      editor.commands.clearContent();
    }
  }

  // Detect and handle variable patterns
  function handleVariableDetection() {
    if (!editor) return;
    
    const { state } = editor;
    const { doc } = state;
    const variablePattern = /{{\s*([^|}]+)(?:\|[^}]*)?\s*}}/g;
    
    doc.descendants((node, pos) => {
      if (node.isText && node.text) {
        const text = node.text;
        let match;
        
        while ((match = variablePattern.exec(text)) !== null) {
          const variableName = match[1].trim();
          const fullMatch = match[0];
          
          // Check if this variable should be converted to inline input
          if (shouldConvertToInlineInput(variableName)) {
            // Replace with inline input
            insertInlineVariableInput(variableName);
            return false; // Stop processing this node
          }
        }
      }
    });
  }

  // Determine if a variable should be converted to inline input
  function shouldConvertToInlineInput(variableName: string): boolean {
    // Convert common variables to inline input
    const inlineVariables = [
      'CLIPBOARD', 'USER_NAME', 'USER_LOCATION', 'USER_LANGUAGE',
      'CURRENT_DATE', 'CURRENT_TIME', 'CURRENT_DATETIME', 'CURRENT_TIMEZONE', 'CURRENT_WEEKDAY'
    ];
    
    return !inlineVariables.includes(variableName);
  }

  onMount(() => {
    editor = new Editor({
      element,
      extensions: [StarterKit],
      content,
      autofocus,
      editorProps: {
        attributes: {
          class: 'tiptap-editor',
          placeholder,
        },
        handleDOMEvents: {
          keydown: (view, event) => {
            // Call the parent's keydown handler
            onKeydown(event);
            return false; // Let the event bubble up
          },
          input: (view, event) => {
            // Detect variable patterns on input
            setTimeout(() => {
              handleVariableDetection();
            }, 100);
            return false;
          }
        }
      },
      onUpdate: ({ editor }) => {
        dispatch('update', { html: editor.getHTML(), text: editor.getText() });
      },
    });
  });

  onDestroy(() => {
    if (editor) editor.destroy();
  });

  // Expose a method to get the current content
  export function getHTML() {
    return editor?.getHTML() ?? '';
  }
  export function getText() {
    return editor?.getText() ?? '';
  }
</script>

<div bind:this={element} class="tiptap-editor-container" />

<style>
.tiptap-editor-container {
  min-height: 2.5em;
  border-radius: 1em;
  background: transparent;
  padding: 0.5em 1em;
  border: 1px solid #e5e7eb;
  font-size: 1em;
  outline: none;
}
.tiptap-editor {
  outline: none;
  min-height: 2.5em;
}

:global(.inline-variable-input) {
  display: inline-block;
  border: 1px solid #3b82f6;
  border-radius: 4px;
  padding: 2px 6px;
  margin: 0 2px;
  font-size: inherit;
  background: #eff6ff;
  color: #1e40af;
  min-width: 80px;
  outline: none;
}

:global(.inline-variable-input:focus) {
  border-color: #1d4ed8;
  background: #dbeafe;
}
</style> 