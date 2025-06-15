<script lang="ts">
  import { onMount } from 'svelte';
  import { writable } from 'svelte/store';

  const API_URL = 'https://684e9ae2f0c9c9848d28973f.mockapi.io/api/notes';
  const PAGE_SIZE = 20;

  let loading = true;
  let error = '';
  let notes: any[] = [];
  let totalNotes = 0;
  let page = 1;

  let newTitle = '';
  let newContent = '';
  let editingId: string | null = null;
  let editTitle = '';
  let editContent = '';
  let deleteId: string | null = null;
  let search = '';
  let sort = 'desc';

  const dark = writable(false);
  onMount(() => {
    dark.set(localStorage.getItem('dark') === 'true');
    dark.subscribe(v => {
      document.documentElement.classList.toggle('dark', v);
      localStorage.setItem('dark', v ? 'true' : 'false');
    });
  });

  async function loadNotes() {
    loading = true;
    error = '';
    try {
      const params = new URLSearchParams({
        page: String(page),
        limit: String(PAGE_SIZE),
        sortBy: 'createdAt',
        order: sort,
        ...(search ? { title: search } : {})
      });
      const res = await fetch(`${API_URL}?${params}`);
      notes = await res.json();
      totalNotes = Number(res.headers.get('X-Total-Count')) || notes.length;
    } catch (e) {
      error = 'Failed to load notes';
      notes = [];
    }
    loading = false;
  }

  onMount(loadNotes);

  async function addNote() {
    if (!newTitle || !newContent) return;
    const tempNote = {
      id: 'temp-' + Math.random(),
      title: newTitle,
      content: newContent,
      createdAt: new Date().toISOString()
    };
    notes = [tempNote, ...notes];
    newTitle = '';
    newContent = '';
    try {
      const res = await fetch(API_URL, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(tempNote)
      });
      const note = await res.json();
      notes = notes.map(n => n.id === tempNote.id ? note : n);
      await loadNotes();
    } catch {
      error = 'Failed to add note';
      notes = notes.filter(n => n.id !== tempNote.id);
    }
  }

  function startEdit(note: any) {
    editingId = note.id;
    editTitle = note.title;
    editContent = note.content;
  }
  async function saveEdit(id: string) {
    try {
      await fetch(`${API_URL}/${id}`, {
        method: 'PUT',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ title: editTitle, content: editContent })
      });
      editingId = null;
      await loadNotes();
    } catch {
      error = 'Failed to edit note';
    }
  }

  async function confirmDelete() {
    if (!deleteId) return;
    const id = deleteId;
    deleteId = null;
    const oldNotes = notes;
    notes = notes.filter(n => n.id !== id);
    try {
      await fetch(`${API_URL}/${id}`, { method: 'DELETE' });
      await loadNotes();
    } catch {
      error = 'Failed to delete note';
      notes = oldNotes;
    }
  }

  function onSearch(e: Event) {
    // @ts-ignore
    search = e.target.value;
    page = 1;
    loadNotes();
  }
  function onSortChange(e: Event) {
    // @ts-ignore
    sort = e.target.value;
    loadNotes();
  }
  function prevPage() {
    if (page > 1) {
      page--;
      loadNotes();
    }
  }
  function nextPage() {
    if (page * PAGE_SIZE < totalNotes) {
      page++;
      loadNotes();
    }
  }
</script>

<!-- DARK MODE TOGGLE -->
<div class="fixed top-4 right-4 z-50">
  <button
    class="rounded-full bg-neutral-200 dark:bg-neutral-800 p-2 shadow hover:scale-110 transition"
    aria-label="Toggle dark mode"
    on:click={() => dark.update(v => !v)}
  >
    <svg class="w-6 h-6" fill="none" stroke="currentColor" viewBox="0 0 24 24">
      <path
        stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
        d="M12 3v1m0 16v1m8.66-8.66l-.71.71M4.05 19.95l-.71-.71M21 12h-1M4 12H3m16.95 4.05l-.71-.71M6.34 6.34l-.71-.71M17.66 6.34l.71-.71M6.34 17.66l-.71.71"
      />
    </svg>
  </button>
</div>

<div class="min-h-screen bg-neutral-100 dark:bg-neutral-900 transition-colors">
  <div class="max-w-xl mx-auto py-10 px-4">
    <h1 class="text-3xl font-semibold text-neutral-800 dark:text-neutral-100 mb-6 text-center tracking-tight">
      Notes
    </h1>

    <!-- SEARCH & SORT -->
    <div class="flex flex-col md:flex-row gap-2 mb-4 items-center justify-between">
      <input
        class="border border-neutral-300 dark:border-neutral-700 bg-neutral-50 dark:bg-neutral-900 text-neutral-900 dark:text-neutral-100 p-2 rounded w-full md:w-1/2"
        placeholder="Search notes by title..."
        value={search}
        on:input={onSearch}
      />
      <select
        class="border border-neutral-300 dark:border-neutral-700 bg-neutral-50 dark:bg-neutral-900 text-neutral-900 dark:text-neutral-100 p-2 rounded"
        value={sort}
        on:change={onSortChange}
      >
        <option value="desc">Newest first</option>
        <option value="asc">Oldest first</option>
      </select>
    </div>

    <!-- Add Note Form -->
    <form
      class="mb-8 flex flex-col gap-3 bg-white dark:bg-neutral-800 rounded-lg shadow p-5"
      on:submit|preventDefault={addNote}
    >
      <input
        class="border border-neutral-300 dark:border-neutral-700 bg-neutral-50 dark:bg-neutral-900 text-neutral-900 dark:text-neutral-100 p-2 rounded focus:outline-none focus:ring-2 focus:ring-blue-500"
        placeholder="Title"
        bind:value={newTitle}
        required
        maxlength={60}
      />
      <textarea
        class="border border-neutral-300 dark:border-neutral-700 bg-neutral-50 dark:bg-neutral-900 text-neutral-900 dark:text-neutral-100 p-2 rounded focus:outline-none focus:ring-2 focus:ring-blue-500"
        placeholder="Content"
        bind:value={newContent}
        required
        rows="3"
        maxlength={400}
      ></textarea>
      <button
        class="self-end bg-neutral-800 dark:bg-neutral-100 text-neutral-100 dark:text-neutral-900 px-5 py-2 rounded transition-colors hover:bg-blue-700 hover:text-white dark:hover:bg-blue-400 dark:hover:text-neutral-900 font-medium"
        type="submit"
      >
        Add Note
      </button>
    </form>

    <!-- LOADING SPINNER -->
    {#if loading}
      <div class="flex justify-center py-8">
        <svg class="animate-spin h-8 w-8 text-blue-500" viewBox="0 0 24 24">
          <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4" fill="none"/>
          <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8v8z"/>
        </svg>
      </div>
    {:else if error}
      <div class="text-red-600 text-center">{error}</div>
    {:else if notes.length === 0}
      <div class="text-center text-neutral-500 dark:text-neutral-400">No notes found.</div>
    {:else}
      <div class="flex flex-col gap-4">
        {#each notes as note}
          <div class="bg-white dark:bg-neutral-800 rounded-lg shadow p-5 flex flex-col gap-2">
            {#if editingId === note.id}
              <input
                class="border border-neutral-300 dark:border-neutral-700 bg-neutral-50 dark:bg-neutral-900 text-neutral-900 dark:text-neutral-100 p-2 rounded"
                bind:value={editTitle}
                maxlength={60}
              />
              <textarea
                class="border border-neutral-300 dark:border-neutral-700 bg-neutral-50 dark:bg-neutral-900 text-neutral-900 dark:text-neutral-100 p-2 rounded"
                bind:value={editContent}
                rows="3"
                maxlength={400}
              ></textarea>
              <div class="flex gap-2 mt-2">
                <button
                  class="bg-blue-700 text-white px-3 py-1 rounded hover:bg-blue-800 transition-colors"
                  on:click={() => saveEdit(note.id)}
                >
                  Save
                </button>
                <button
                  class="bg-neutral-300 dark:bg-neutral-700 text-neutral-800 dark:text-neutral-200 px-3 py-1 rounded hover:bg-neutral-400 dark:hover:bg-neutral-600 transition-colors"
                  on:click={() => editingId = null}
                >
                  Cancel
                </button>
              </div>
            {:else}
              <div class="flex justify-between items-center">
                <h2 class="text-lg font-semibold text-neutral-800 dark:text-neutral-100">{note.title}</h2>
                <span class="text-xs text-neutral-500 dark:text-neutral-400">{new Date(note.createdAt).toLocaleString()}</span>
              </div>
              <p class="text-neutral-700 dark:text-neutral-200">{note.content}</p>
              <div class="flex gap-2 mt-2">
                <button
                  class="bg-neutral-900 dark:bg-neutral-100 text-neutral-100 dark:text-neutral-900 px-3 py-1 rounded hover:bg-blue-700 hover:text-white dark:hover:bg-blue-400 dark:hover:text-neutral-900 transition-colors"
                  on:click={() => startEdit(note)}
                >
                  Edit
                </button>
                <button
                  class="bg-red-600 text-white px-3 py-1 rounded hover:bg-red-700 transition-colors"
                  on:click={() => deleteId = note.id}
                >
                  Delete
                </button>
              </div>
            {/if}
          </div>
        {/each}
      </div>
      <!-- Pagination -->
      <div class="flex justify-center gap-4 mt-6">
        <button
          class="px-3 py-1 rounded bg-neutral-200 dark:bg-neutral-700 text-neutral-700 dark:text-neutral-200 disabled:opacity-50"
          on:click={prevPage}
          disabled={page === 1}
        >Previous</button>
        <span class="self-center text-neutral-600 dark:text-neutral-400">
          Page {page} of {Math.ceil(totalNotes / PAGE_SIZE) || 1}
        </span>
        <button
          class="px-3 py-1 rounded bg-neutral-200 dark:bg-neutral-700 text-neutral-700 dark:text-neutral-200 disabled:opacity-50"
          on:click={nextPage}
          disabled={page * PAGE_SIZE >= totalNotes}
        >Next</button>
      </div>
    {/if}

    <!-- DELETE CONFIRMATION MODAL -->
    {#if deleteId}
      <div class="fixed inset-0 bg-black/40 flex items-center justify-center z-50">
        <div class="bg-white dark:bg-neutral-800 rounded-lg shadow-lg p-6 max-w-sm w-full">
          <h2 class="text-lg font-bold mb-2 text-neutral-800 dark:text-neutral-100">Delete Note</h2>
          <p class="mb-4 text-neutral-700 dark:text-neutral-200">Are you sure you want to delete this note?</p>
          <div class="flex justify-end gap-2">
            <button
              class="bg-neutral-300 dark:bg-neutral-700 text-neutral-800 dark:text-neutral-200 px-4 py-1 rounded hover:bg-neutral-400 dark:hover:bg-neutral-600"
              on:click={() => deleteId = null}
            >Cancel</button>
            <button
              class="bg-red-600 text-white px-4 py-1 rounded hover:bg-red-700"
              on:click={confirmDelete}
            >Delete</button>
          </div>
        </div>
      </div>
    {/if}
  </div>
</div>
