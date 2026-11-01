<script lang="ts">
    import type {GroupedModules, Module as TModule} from "../../integration/types";
    import Module from "./Module.svelte";
    import Search from "./Search.svelte";
    import Description from "./Description.svelte";
    import {fade, scale} from "svelte/transition";
    import {onMount} from "svelte";
    import {getCategories, getModules} from "../../integration/rest";
    import {listen} from "../../integration/ws";
    import type {ModuleToggleEvent} from "../../integration/events";
    import {groupByCategory} from "../../integration/util";
    import {highlightModuleName} from "./clickgui_store";
    import ScaledClickGuiContent from "./ScaledClickGuiContent.svelte";

    let categories = $state<GroupedModules>({});
    let modules = $state<TModule[]>([]);
    let icons = $state<Record<string, string>>({});
    let selected = $state<string>("");

    function showFallbackIcon(event: Event) {
        (event.currentTarget as HTMLImageElement).src = "img/clickgui/icon-client.svg";
    }

    function select(category: string) {
        selected = category;
        try {
            localStorage.setItem("clickgui.rise.category", category);
        } catch {
            // ignore
        }
    }

    onMount(async () => {
        modules = await getModules();
        categories = groupByCategory(modules);
        icons = Object.fromEntries(
            (await getCategories())
                .filter(category => category.icon)
                .map(category => [category.name, category.icon!])
        );

        let saved: string | null = null;
        try {
            saved = localStorage.getItem("clickgui.rise.category");
        } catch {
            // ignore
        }
        const names = Object.keys(categories);
        selected = saved && names.includes(saved) ? saved : names[0] ?? "";
    });

    // Search jumps to the category of the highlighted module
    highlightModuleName.subscribe((name) => {
        for (const [category, list] of Object.entries(categories)) {
            if (list.some(m => m.name === name)) {
                selected = category;
                return;
            }
        }
    });

    listen("moduleToggle", (e: ModuleToggleEvent) => {
        const mod = modules.find(m => m.name === e.moduleName);
        if (mod) {
            mod.enabled = e.enabled;
        }
    });

    function enabledCount(category: string) {
        return (categories[category] ?? []).filter(m => m.enabled).length;
    }
</script>

<ScaledClickGuiContent>
    <div class="clickgui" transition:fade|global={{duration: 200}}>
        <Description/>
        <Search modules={structuredClone($state.snapshot(modules))}/>

        <div class="window" transition:scale|global={{duration: 220, start: 0.96}}>
            <aside class="sidebar">
                <div class="brand">Void<b>Bounce</b></div>
                {#each Object.keys(categories) as category (category)}
                    <button class="category" class:active={category === selected} onclick={() => select(category)}>
                        <img
                                src={icons[category] ?? `img/clickgui/icon-${category.toLowerCase()}.svg`}
                                alt=""
                                onerror={showFallbackIcon}
                        />
                        <span class="label">{category}</span>
                        <span class="count">{enabledCount(category)}</span>
                    </button>
                {/each}
            </aside>

            <section class="content">
                <header>
                    <span class="title">{selected}</span>
                    <span class="subtitle">{(categories[selected] ?? []).length} modules</span>
                </header>
                <div class="modules">
                    {#each categories[selected] ?? [] as {name, enabled, description, aliases} (selected + name)}
                        <Module {name} {enabled} {description} {aliases}/>
                    {/each}
                </div>
            </section>
        </div>
    </div>
</ScaledClickGuiContent>

<style lang="scss">
  .clickgui {
    position: absolute;
    inset: 0;
    display: flex;
    align-items: center;
    justify-content: center;
    user-select: none;
  }

  .window {
    display: flex;
    width: 780px;
    height: 480px;
    margin-top: 40px;
    overflow: hidden;
    border-radius: 14px;
    background-color: rgba(10, 10, 14, 0.62);
    backdrop-filter: blur(14px);
    border: 1px solid rgba(255, 255, 255, 0.08);
    box-shadow: 0 10px 40px rgba(0, 0, 0, 0.55);
  }

  .sidebar {
    width: 190px;
    padding: 14px 10px;
    display: flex;
    flex-direction: column;
    gap: 3px;
    background-color: rgba(0, 0, 0, 0.3);
    border-right: 1px solid rgba(255, 255, 255, 0.06);
    overflow-y: auto;

    .brand {
      padding: 4px 10px 16px;
      font-size: 22px;
      font-weight: 300;
      color: var(--clickgui-text-color);

      b {
        font-weight: 700;
      }
    }
  }

  .category {
    all: unset;
    display: grid;
    grid-template-columns: 18px 1fr auto;
    align-items: center;
    column-gap: 10px;
    padding: 9px 10px;
    border-radius: 8px;
    cursor: pointer;
    font-size: 13px;
    font-weight: 500;
    color: var(--clickgui-text-dimmed-color);
    transition: background-color 0.15s ease, color 0.15s ease;

    img {
      width: 16px;
      height: 16px;
      opacity: 0.7;
    }

    .count {
      font-size: 11px;
      opacity: 0.5;
    }

    &:hover {
      background-color: rgba(255, 255, 255, 0.06);
      color: var(--clickgui-text-color);
    }

    &.active {
      background-color: rgba(255, 255, 255, 0.14);
      color: var(--clickgui-text-color);

      img {
        opacity: 1;
      }
    }
  }

  .content {
    flex: 1;
    min-width: 0;
    display: flex;
    flex-direction: column;

    header {
      display: flex;
      align-items: baseline;
      gap: 10px;
      padding: 16px 20px 12px;
      border-bottom: 1px solid rgba(255, 255, 255, 0.06);

      .title {
        font-size: 18px;
        font-weight: 600;
        color: var(--clickgui-text-color);
      }

      .subtitle {
        font-size: 12px;
        color: var(--clickgui-text-dimmed-color);
        opacity: 0.6;
      }
    }
  }

  .modules {
    flex: 1;
    overflow-y: auto;
    padding: 10px 14px;

    &::-webkit-scrollbar {
      width: 3px;
    }

    &::-webkit-scrollbar-thumb {
      border-radius: 3px;
      background-color: rgba(255, 255, 255, 0.2);
    }
  }
</style>
