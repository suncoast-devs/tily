<script lang="ts">
  import { writable } from 'svelte/store'

  const STEP_DELAY = 3200
  const COUNT_VALUE = '{N}'

  let editing = writable(true)
  let running = writable(false)
  let cursor = 0
  let timer: ReturnType<typeof setInterval>

  let errorLine: number | null
  let errorMessage: string | null

  let retVal: string | null

  const program = writable(
    '{ABCDE}\nALLOCATE position\nASSIGN position 0\nALLOCATE max\nASSIGN max {N}\nALLOCATE letter\n#LOOP_BEGIN\nREAD_TILE position letter\nINCREMENT position\nJUMP_IF_EQUAL position max #EXIT\nJUMP #LOOP_BEGIN\n#EXIT\nSTOP YES'
  )

  interface Instruction {
    instruction: string
    args: (string | number)[]
  }

  interface Memory {
    [key: string]: any
  }

  let tiles: string[] = []

  let memory: Memory = {}
  let tags: Memory = {}
  let instructions: Instruction[] = []

  function parse() {
    instructions = []
    $program.split(/\r?\n/).forEach((line) => {
      if (/^{([\w\s]*)}$/.test(line)) {
        tiles = line.match(/[A-Z]/g) || []
      } else if (line.startsWith('#')) {
        tags[line] = instructions.length
        instructions.push({ instruction: 'TAG', args: [line, instructions.length] })
      } else {
        const [instruction, ...args] = line.split(' ')
        instructions.push({
          instruction,
          args: args.map((a) => (/^\d+$/.test(a) ? parseInt(a) : a)),
        })
      }
    })
  }

  function run() {
    // Reset
    errorLine = null
    errorMessage = null
    tiles = []
    memory = {}
    cursor = 0

    $running = true
    parse()
    doCurrentInstruction()
    // }, STEP_DELAY)
  }

  function stop() {
    if (timer) clearInterval(timer)
    $running = false
  }

  function doCurrentInstruction() {
    if (cursor > instructions.length) {
      stop()
      return
    }

    const inst = instructions[cursor]
    if (!inst) return

    switch (inst.instruction) {
      case 'ALLOCATE':
        // TODO: Error checking; duplicates
        memory[inst.args[0]] = null
        break
      case 'ASSIGN':
        const value = inst.args[1]
        memory[inst.args[0]] = value === COUNT_VALUE ? tiles.length : value
        break
      case 'TAG':
        if ((inst.args[0] as string).length < 2) showError(`No label for tag, e.g. "#TAG_NAME"`)
        if (memory[inst.args[0]]) showError(`Duplicate tag "${inst.args[0]}"`)
        break
      case 'INCREMENT':
        // TODO: Error checking; variable exists
        memory[inst.args[0]] = memory[inst.args[0]] + 1
        break
      case 'JUMP_IF_EQUAL':
        // TODO: Error checking; tag exists, etc.
        if (memory[inst.args[0]] === memory[inst.args[1]]) {
          cursor = tags[inst.args[2]]
        }
        break
      case 'JUMP':
        // TODO: Error checking; tag exists, etc.
        cursor = tags[inst.args[0]]
        break
      case 'READ_TILE':
        const position = memory[inst.args[0]]
        memory[inst.args[1]] = tiles[position]
        break
      case 'STOP':
        // TODO: Error checking; YES or NO
        retVal = inst.args[0] as string
        break
      default:
        showError(`Unknown Instruction "${inst.instruction}"`)
        break
    }

    // Skip over tags with less (or no) delay
    setTimeout(() => doCurrentInstruction(), STEP_DELAY)
    setTimeout(() => cursor++, STEP_DELAY / 2)
  }

  function showError(message: string) {
    errorLine = cursor
    errorMessage = message
    stop()
  }

  function closeEditor() {
    $editing = false
    run()
  }

  function openEditor() {
    $editing = true
    stop()
  }

  function colorForArg(arg: any) {
    if (/^\d+$/.test(arg) || arg === COUNT_VALUE) {
      return 'text-amber-500'
    } else if (/^#/.test(arg)) {
      return 'text-fuchsia-400'
    } else {
      return 'text-emerald-600'
    }
  }
</script>

<svelte:head>
  <title>Tily</title>
</svelte:head>

<div class="mx-auto container px-2">
  <nav class="py-2 flex justify-between">
    <h1 class="text-4xl font-bold text-slate-800">Tily</h1>
    <div>
      {#if $editing}
        <button class="bg-slate-500 rounded-md text-slate-100 p-2" on:click={closeEditor}
          >RUN</button
        >
      {:else}
        {#if !$running}
          <button class="bg-slate-500 rounded-md text-slate-100 p-2" on:click={run}>RE-RUN</button>
        {/if}
        <button class="bg-slate-500 rounded-md text-slate-100 p-2" on:click={openEditor}
          >EDIT</button
        >
      {/if}
    </div>
  </nav>

  <div class="grid gap-4 md:grid-cols-2">
    <div class="space-y-4">
      {#if $editing}
        <div class="border-solid border-2 border-amber-500 rounded-md bg-amber-50 ">
          <h2 class="bg-amber-500 text-amber-800 font-bold p-2">Source Code</h2>
          <textarea
            bind:value={$program}
            cols="32"
            rows="20"
            class="p-2 font-mono text-slate-800 w-full bg-amber-50"
          />
        </div>
      {:else}
        <div class="border-solid border-2 border-slate-400 rounded-md bg-slate-200 text-slate-600">
          <h2 class="bg-slate-400 text-slate-100 font-bold p-2">Program</h2>
          {#each instructions as inst, i}
            <div
              class="p-2 border-y-2 first:border-t-0 last:border-b-0 font-mono {$running &&
              i === cursor
                ? 'bg-emerald-200 border-y-emerald-300'
                : 'border-y-slate-200'}
                
              {i === errorLine ? 'bg-red-200 border-y-red-300' : ''}"
            >
              <span class="{i === cursor ? 'text-emerald-400' : 'text-slate-300'} float-right"
                >{i + 1}</span
              >
              {#if inst.instruction === 'TAG'}
                <span class="text-fuchsia-400">{inst.args[0]}</span>
              {:else}
                <span class="text-cyan-600">{inst.instruction}</span>
                {#each inst.args as arg}
                  <span class={colorForArg(arg)}>{arg}</span>&nbsp;
                {/each}
              {/if}
            </div>
          {/each}
        </div>
        {#if retVal}
          <div
            class="p-2 inline-block rounded-md border-2 {retVal === 'YES'
              ? 'bg-emerald-300 text-emerald-700 border-emerald-500'
              : 'bg-amber-300 text-amber-700 border-amber-500'}"
          >
            <code>EXIT: {retVal}</code>
          </div>
        {/if}
      {/if}
    </div>
    <div class="space-y-4">
      {#if $editing}
        <ul class="space-y-4">
          <li>
            You can defined the tiles available to your program like this: <code class="font-bold"
              >{'{ASDFQ}'}</code
            >.
          </li>
          <li>
            You can tag locations in your program like this: <code class="font-bold"
              >#[TAG_NAME]</code
            >. Later you can use the JUMP functions to control the flow of the program and return to
            these tagged lines.
          </li>
          <li>
            <code class="font-bold">ALLOCATE [name]</code>: Allocate a new memory location
            referenced by [name]. Don't use square brackets in your names. These are just
            placeholders for your own labels in your progam.
          </li>
          <li>
            <code class="font-bold">ASSIGN [value]</code>: Assign the given value to the named
            location you've already allocated.
          </li>
          <li>
            <code class="font-bold">READ_TILE [location] [destination]</code>: Read the tile at the
            given location and copy it to the destination. The tile is read by it's position from
            left-to-right starting with 0. The [location] is a memory location that contains a
            number for the tile's position. The [destination] is a memory location that will contain
            the tile's value. If this one is tricky to understand; look at the example program, it
            should help.
          </li>
          <li>
            <code class="font-bold">INCREMENT [name]</code>: Increment the value at the named
            location by 1. The value must be a number.
          </li>
          <li>
            <code class="font-bold">JUMP_IF_EQUAL [name] [name] #[TAG]</code>: Jump to the point in
            the program with the given <code>#TAG</code> if the values at both named locations are equal.
          </li>
          <li>
            <code class="font-bold">JUMP #[TAG]</code>: Jump to the point in the program with the
            given <code>#TAG</code>.
          </li>
          <li>
            <code class="font-bold">EXIT [YES | NO]</code>: Exit the program with a YES or NO return
            value.
          </li>
        </ul>
      {:else}
        <div
          class="flex justify-start flex-wrap border-stone-300 bg-stone-100 border-2 p-1 rounded-md"
        >
          {#each tiles as tile}
            <div
              class="border-solid border-2 m-1 border-stone-400 rounded-md bg-stone-200 text-stone-500 w-10 h-10 font-bold text-2xl flex justify-center items-center"
            >
              <span>{tile}</span>
            </div>
          {/each}
        </div>
        <div class="border-sky-400 border-2 bg-sky-200 rounded-md">
          <h2 class="bg-sky-400 font-bold text-sky-700 p-2">Memory</h2>
          <table class="table-auto w-full text-sky-700">
            <tbody>
              {#each Object.entries(memory) as [k, v]}
                <tr>
                  <td class="p-2 bg-sky-300 w-1/2">
                    <kbd>{k}</kbd>
                  </td>
                  <td class="p-2 w-1/2">
                    <code>{v}</code>
                  </td>
                </tr>
              {/each}
            </tbody>
          </table>
        </div>

        {#if errorMessage}
          <div class="border-red-400 border-2 bg-red-200 rounded-md  text-red-700">
            <h2 class="bg-red-400 font-bold text-red-900 p-2">ERROR!</h2>
            <p class="p-2">{errorMessage}</p>
          </div>
        {/if}
      {/if}
    </div>
  </div>
</div>
