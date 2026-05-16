<script setup lang="ts">
import { type Ref, ref } from "vue";

import UiButton from "@/components/ui/Button.vue";
import PoseBar from "@/components/ui/PoseBar.vue";
import ShortBar from "@/components/ui/ShortBar.vue";
import SideBar from "@/components/ui/SideBar.vue";

import AnimationMultiPlayer from "@/components/common/AnimationMultiPlayer.vue";
import AnimationPlayer from "@/components/common/AnimationPlayer.vue";
import ColorSelect from "@/components/common/ColorSelect.vue";
import CreditsCard from "@/components/common/CreditsCard.vue";
import RenderingsCard from "@/components/common/RenderingsCard.vue";
import SavingsCard from "@/components/common/SavingsCard.vue";
import SpriteCard from "@/components/common/SpriteCard.vue";

import SpriteCanvas from "@/components/character/SpriteCanvas.vue";

import { CharacterRenderer } from "@/services/CharacterRenderer";

import { CharacterCollection } from "@/types/CharacterCollection";
import { ColorCollection } from "@/types/ColorCollection";
import { Item } from "@/types/Item";

import raw_colors from '@/data/colors.json';
import raw_data from '@/data/packed.json';

import JSZip from 'jszip';

const colors:Ref<ColorCollection> = ref(new ColorCollection())
colors.value.initColors(raw_colors)
const collection:Ref<CharacterCollection> = ref(new CharacterCollection(colors.value)) as Ref<CharacterCollection>
collection.value.initItems(raw_data);
const renderer:CharacterRenderer = new CharacterRenderer(collection.value);

const spriteCanvas = ref()
const refresh:Ref<number> = ref(0)
const sideBar:Ref<any> = ref()

const leftTab:Ref<string> = ref('anatomy');
const rightTab:Ref<string> = ref('savings');
const centerTab:Ref<string> = ref('preview-multiple');

const currentColorPicker:Ref<boolean> = ref(false)
const currentColorItem:Ref<Item | null> = ref(null)
const currentColorMaterial:Ref<string> = ref('')

function onToggleColorSelector(item:Item, material:string) {
  currentColorPicker.value = !!item;
  currentColorItem.value = item;
  currentColorMaterial.value = material;
}

async function onColorSelected(color:string) {
  if(!currentColorItem.value) {
    return
  }

  currentColorItem.value?.colors.set(currentColorMaterial.value, color);
  await collection.value.select(currentColorItem.value)
  renderer.draw();
  refresh.value++;

  onColorPickerClosed();
}

function onColorPickerClosed() {
  currentColorPicker.value = false
  currentColorItem.value = null
  currentColorMaterial.value = ''

  sideBar.value.$el.scrollTo(0, 0)
}

function jumpToNextActiveAnimation() {
  if(collection.value.isAnimationDisabled(currentPose.value)) {
    currentPose.value = 'walk'
  }
}

async function onRefresh() {
  renderer.draw();
  refresh.value++;
  jumpToNextActiveAnimation()
}

function getCredits():{[key:string]:{[key:string]:any}} {
	return collection.value.getCredits()
}

async function generateRandomCharacter() {
	// Reset to base state first
	await collection.value.reset()

	while (!(await _generateRandomCharacter().catch(() => false)));

	// Call the onDownloadAll function to trigger the download of the generated character's spritesheets and credits
	try {
		await (async () => {
			const zip = new JSZip();

			const animations = renderer.getAnimationCanvases();

			for(const animation in animations) {
				zip.file(animation + '.png', animations[animation].value.toDataURL().split(';base64,').pop(), { base64: true });
			}

			zip.file('credits.json', JSON.stringify(getCredits(), null, 2));

			const content:any = await zip.generateAsync({ type: 'blob' });
			const blob:Blob = new Blob([content], { type: 'application/zip' });
			const link:HTMLAnchorElement = document.createElement('a');
			link.href = URL.createObjectURL(blob);
			link.download = `spritesheets-${Date.now()}.zip`;
			link.click();
		})();
	} catch (e) {
		console.warn('onDownloadAll failed', e)
	}
}

async function _generateRandomCharacter() {
  const categories = collection.value.getSpriteCategories();

  function getAllColorKeys() {
    return Object.keys(colors.value.getAll())
  }

  function pickColorFromPaletteCandidates(candidates:string[]):string {
    const allColors = colors.value.getAll();

    // Try direct key match first
    for(const c of candidates) {
      if(allColors[c]) {
        return c
      }
    }

    // Treat candidate as material category (e.g. "skin", "hair", "any")
    const materialCandidates:string[] = []

    for(const key in allColors) {
      const col = allColors[key]
      for(const cand of candidates) {
        if(col.materials && (col.materials.indexOf(cand) >= 0 || col.materials.indexOf('any') >= 0)) {
          materialCandidates.push(key)
          break
        }
      }
    }

    if(materialCandidates.length) {
      return materialCandidates[Math.floor(Math.random() * materialCandidates.length)]
    }

    // Fallback to any color
    const keys = getAllColorKeys()
    return keys[Math.floor(Math.random() * keys.length)]
  }

  function includeProbabilityFor(type:string):number {
    if(!type) return 0.35

    if(type.endsWith('.body') || type.endsWith('.head')) {
      return 1.0
    }

    if(type.startsWith('clothes.head_coverings') || type.startsWith('clothes.helmet_accessory')) {
      return 0.2
    }

    if(type.startsWith('equipment.shields')) {
      return 0.25
    }

    if(type.startsWith('injuries.')) {
      return 0.15
    }

    if(type.startsWith('clothes.')) {
      return 0.35
    }

    return 0.35
  }

  for(const catKey in categories) {
    const cat = (categories as any)[catKey]

    // handle flat children
    if(cat.children) {
      // Special handling for equipment: pick at most one weapon/tool
      if(catKey === 'equipment') {
        const weaponTypes = new Set([
          'equipment.bows',
          'equipment.bow_accessory',
          'equipment.spears',
          'equipment.staffs',
          'equipment.staff_accessory',
          'equipment.swords',
          'equipment.tools',
          'equipment.misc'
        ])

        const weaponChildren = cat.children.filter((c:any) => weaponTypes.has(c.type))
        const otherChildren = cat.children.filter((c:any) => !weaponTypes.has(c.type))

        // Select all non-weapon equipment children as usual (with chance to be none)
        for(const child of otherChildren) {
          const options:{[key:string]:any} = collection.value.getOptions(child.type)

          const keys = Object.keys(options)
          if(!keys.length) {
            continue
          }

          const prob = includeProbabilityFor(child.type)
          if(Math.random() > prob) {
            continue
          }

          const pick = options[keys[Math.floor(Math.random() * keys.length)]]

          for(const material in pick.materials) {
            const mats = pick.materials[material].palettes || []
            if(mats.length) {
              const chosen = pickColorFromPaletteCandidates(mats)
              pick.colors.set(material, chosen)
            }
          }

          await collection.value.select(pick)
        }

        // For weapon/tool children, pick at most one (weighted chance to pick none)
        if(weaponChildren.length) {
          const includeProbability = 0.25 // probability to include a weapon/tool (25%)
          if(Math.random() < includeProbability) {
            const child = weaponChildren[Math.floor(Math.random() * weaponChildren.length)]
            const options:{[key:string]:any} = collection.value.getOptions(child.type)

            const keys = Object.keys(options)
            if(keys.length) {
              const pick = options[keys[Math.floor(Math.random() * keys.length)]]

              for(const material in pick.materials) {
                const mats = pick.materials[material].palettes || []
                if(mats.length) {
                  const chosen = pickColorFromPaletteCandidates(mats)
                  pick.colors.set(material, chosen)
                }
              }

              await collection.value.select(pick)
            }
          }
        }
      } else {
        for(const child of cat.children) {
          const options:{[key:string]:any} = collection.value.getOptions(child.type)

          const keys = Object.keys(options)
          if(!keys.length) {
            continue
          }

          const prob = includeProbabilityFor(child.type)
          if(Math.random() > prob) {
            continue
          }

          const pick = options[keys[Math.floor(Math.random() * keys.length)]]

          // assign random palettes for materials
          for(const material in pick.materials) {
            const mats = pick.materials[material].palettes || []
            if(mats.length) {
              const chosen = pickColorFromPaletteCandidates(mats)
              pick.colors.set(material, chosen)
            }
          }

          await collection.value.select(pick)
        }
      }
    }

    // handle tabs (clothes has tabs)
    if(cat.tabs) {
      for(const tabKey in cat.tabs) {
        const tab = cat.tabs[tabKey]
        for(const child of tab.children) {
          const options:{[key:string]:any} = collection.value.getOptions(child.type)

          const keys = Object.keys(options)
          if(!keys.length) {
            continue
          }

          const prob = includeProbabilityFor(child.type)
          if(Math.random() > prob) {
            continue
          }

          const pick = options[keys[Math.floor(Math.random() * keys.length)]]

          for(const material in pick.materials) {
            const mats = pick.materials[material].palettes || []
            if(mats.length) {
              const chosen = pickColorFromPaletteCandidates(mats)
              pick.colors.set(material, chosen)
            }
          }

          await collection.value.select(pick)
        }
      }
    }
  }

  renderer.draw()
  refresh.value++
  jumpToNextActiveAnimation()

  return true
}

const currentPose = ref('walk')
</script>

<template>
  <section class="flex flex-1 overflow-hidden relative bg-slate-900 divide-x divide-slate-800">
    <short-bar>
      <ui-button @click="leftTab = 'anatomy'" :active="leftTab == 'anatomy'" ui="primary-square" title="Anatomy" icon="human"></ui-button>
      <ui-button @click="leftTab = 'clothes'" :disabled="!collection.isBodySelected()" :active="leftTab == 'clothes'" ui="primary-square" title="Clothes" icon="hanger"></ui-button>
      <ui-button @click="leftTab = 'injuries'" :disabled="!collection.isBodySelected()" :active="leftTab == 'injuries'" ui="primary-square" title="Injuries" icon="account-injury"></ui-button>
      <ui-button @click="leftTab = 'equipment'" :disabled="!collection.isBodySelected()" :active="leftTab == 'equipment'" ui="primary-square" title="Equipment" icon="axe"></ui-button>
    </short-bar>

    <side-bar ref="sideBar">
      <sprite-card @selected="onRefresh" @toggle-color-selector="onToggleColorSelector" :hidden="currentColorPicker" :refresh="refresh" :collection="collection" :tab="leftTab"></sprite-card>
      <color-select @selected="onColorSelected" @close="onColorPickerClosed" :collection="collection" :item="currentColorItem" :material="currentColorMaterial" :class="{hidden: !currentColorPicker}" class="bg-slate-900 text-slate-400"></color-select>
    </side-bar>

    <main class="flex flex-col overflow-hidden flex-1 gap-4 bg-slate-800">
      <pose-bar :class="{hidden: centerTab == 'sprites'}">
        <ui-button :disabled="collection.isAnimationDisabled('walk')" @click="currentPose = 'walk'" :active="currentPose == 'walk'" ui="primary" title="Walk">Walk</ui-button>
        <ui-button :disabled="collection.isAnimationDisabled('shoot')" @click="currentPose = 'shoot'" :active="currentPose == 'shoot'" ui="primary" title="Shoot">Shoot</ui-button>
        <ui-button :disabled="collection.isAnimationDisabled('slash')" @click="currentPose = 'slash'" :active="currentPose == 'slash'" ui="primary" title="Slash">Slash</ui-button>
        <ui-button :disabled="collection.isAnimationDisabled('backslash')" @click="currentPose = 'backslash'" :active="currentPose == 'backslash'" ui="primary" title="Backslash">Backslash</ui-button>
        <ui-button :disabled="collection.isAnimationDisabled('spell')" @click="currentPose = 'spell'" :active="currentPose == 'spell'" ui="primary" title="Spell">Spell</ui-button>
        <ui-button :disabled="collection.isAnimationDisabled('thrust')" @click="currentPose = 'thrust'" :active="currentPose == 'thrust'" ui="primary" title="Thrust">Thrust</ui-button>
        <ui-button :disabled="collection.isAnimationDisabled('rod')" @click="currentPose = 'rod'" :active="currentPose == 'rod'" ui="primary" title="Rod">Rod</ui-button>
        <ui-button :disabled="collection.isAnimationDisabled('whip')" @click="currentPose = 'whip'" :active="currentPose == 'whip'" ui="primary" title="Whip">Whip</ui-button>
        <ui-button :disabled="collection.isAnimationDisabled('hurt')" @click="currentPose = 'hurt'" :active="currentPose == 'hurt'" ui="primary" title="Hurt">Hurt</ui-button>
        <ui-button :disabled="collection.isAnimationDisabled('idle')" @click="currentPose = 'idle'" :active="currentPose == 'idle'" ui="primary" title="Idle">Idle</ui-button>

      </pose-bar>
      <div class="grow overflow-scroll scrollbar-thin p-2">
        <sprite-canvas ref="spriteCanvas" :current="currentPose" :renderer="renderer" :class="{'hidden': centerTab != 'sprites'}"></sprite-canvas>
        <animation-multi-player :current="currentPose" :collection="collection" :renderer="renderer" :class="{'hidden': centerTab != 'preview-multiple'}"></animation-multi-player>
        <animation-player :current="currentPose" :collection="collection" :renderer="renderer" :class="{'hidden': centerTab != 'preview'}"></animation-player>
      </div>
      <div class="flex gap-2 justify-center h-10 max-h-10 grow-0 shrink-0">
        <ui-button @click="centerTab = 'preview-multiple'" :active="centerTab == 'preview-multiple'" ui="primary-square" title="Preview Multiple" icon="filmstrip-box-multiple"></ui-button>
        <ui-button @click="centerTab = 'preview'" :active="centerTab == 'preview'" ui="primary-square" title="Preview" icon="filmstrip-box"></ui-button>
        <ui-button @click="centerTab = 'sprites'" :active="centerTab == 'sprites'" ui="primary-square" title="Sprites" icon="grid"></ui-button>
        <ui-button @click="generateRandomCharacter()" ui="primary-square" title="Random" icon="magic-staff" id="generate-random"></ui-button>
      </div>
    </main>

    <side-bar>
      <savings-card @loaded="onRefresh" :collection="collection" :renderer="renderer" type="character" :class="{'hidden': rightTab != 'savings'}"></savings-card>
      <credits-card :collection="collection" :class="{'hidden': rightTab != 'credits'}"></credits-card>
      <renderings-card :collection="collection" :renderer="renderer" :current="currentPose" :class="{'hidden': rightTab != 'render'}"></renderings-card>
    </side-bar>
    <short-bar>
      <ui-button @click="rightTab = 'savings'" :active="rightTab == 'savings'" ui="primary-square" title="Savings" icon="folder"></ui-button>
      <ui-button :disabled="!collection.isBodySelected()" @click="rightTab = 'credits'" :active="rightTab == 'credits'" ui="primary-square" title="Credits" icon="license"></ui-button>
      <ui-button :disabled="!collection.isBodySelected()" @click="rightTab = 'render'" :active="rightTab == 'render'" ui="primary-square" title="Renderings" icon="folder-image"></ui-button>
    </short-bar>
  </section>
</template>

<style scoped>

</style>