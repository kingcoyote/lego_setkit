<script lang="ts" setup>

import { useLocalStorage, useBreakpoints, breakpointsVuetifyV3 } from "@vueuse/core"

interface I_SET {
  name: string,
  set_num: number,
  set_img_url: string,
  year: number,
  num_parts: number,
}

interface I_PART {
  quantity: number,
  part: {
    part_num: number,
    name: string,
    part_img_url: string,
  }
  color: {
    id: number,
    name: string
  }
}

const api_secret = useRuntimeConfig().public.rebrickableApiSecret;

const breakpoints = useBreakpoints(breakpointsVuetifyV3)

const set = useLocalStorage<I_SET | null>('setkit.set', null);
const parts = useLocalStorage<I_PART[]>('setkit.parts', [])
const active_set_id = useLocalStorage<number>('setkit.active_set', -1);

let kit = ref<Record<string, number>>({});
if (active_set_id.value > 0) {
  kit = useLocalStorage(`setkit.kit.${active_set_id.value}`, {});
}
const showPicked = ref(false)

const set_list = [
  60378,
  30161,
  76177,
  41737,
  31138,
  71363,
  60383,
  10786,
  10788,
  77092,
  40721,
  21330,
  40599,
  77049,
  77046,
  77048,
  40429,
  40346,
  21047,
  43213,
  71454,
  31120,
  10696,
  21309,
  40519,
  31147,
  31134,
  21251,
  10790,
  21166,
  21241,
  21546,
  21242,
  21179,
  21170,
  21152,
  21189,
  21240,
]

async function searchForSet() {
  set.value = {}
  parts.value = []
  kit.value = {}

  let response = await fetch(`https://rebrickable.com/api/v3/lego/sets/${active_set_id.value}-1/?key=${api_secret}`);
  set.value = await response.json()

  let next = `https://rebrickable.com/api/v3/lego/sets/${active_set_id.value}-1/parts?key=${api_secret}`

  while (next != null) {
    response = await fetch(next);
    let result = await response.json();
    parts.value = parts.value.concat(result['results'].filter(p => p.is_spare === false));
    next = result['next'];
  }
}

function loadRandomSet() {
  active_set_id.value = set_list[Math.floor(Math.random() * set_list.length)];
  searchForSet();
}

function keyToPart(part_key: string) : I_PART | undefined {
  const part_keys = part_key.split('-').map((x) => parseInt(x))
  return parts.value.find((p) => p.part.part_num == part_keys[0] && p.color.id == part_keys[1])
}

function completePart(part_key: string) {
  const part = keyToPart(part_key)
  if (part !== undefined)
    kit.value[part_key] = part.quantity;
}

function incrementPart(part_key: string) {
  if (kit.value[part_key] === undefined) {
    kit.value[part_key] = 0;
  }

  kit.value[part_key] += 1;

  const part = keyToPart(part_key);
  if (part !== undefined && kit.value[part_key] > part.quantity) {
    kit.value[part_key] = part.quantity;
  }
}

function decrementPart(part_key: string) {
  if (kit.value[part_key] === undefined) {
    kit.value[part_key] = 0;
  }

  kit.value[part_key] -= 1;

  if (kit.value[part_key] < 0) {
    kit.value[part_key] = 0;
  }
}

function highlightPart(part_key: string) {
  console.log(part_key)
}

const kitted_parts = computed(() => {
  const all_parts = Object.values(kit.value)
  if (all_parts.length == 0) return 0;
  return all_parts.reduce((a, b) => a + b);
})

const debug = ref(false)
const drawNavigation = ref(true);

</script>

<template>
  <v-layout>
  <v-app-bar>
    <template v-slot:prepend>
      <v-app-bar-nav-icon @click.stop="drawNavigation = !drawNavigation"></v-app-bar-nav-icon>
    </template>
    <v-app-bar-title>Lego Setkit</v-app-bar-title>
    <v-spacer></v-spacer>
  </v-app-bar>
  <v-navigation-drawer v-model="drawNavigation" :location="breakpoints.smaller('sm').value ? 'top' : 'left'">
    <v-list>
      <v-list-item link title="Search for Set" @click="searchForSet"/>
      <v-list-item><v-text-field label="Set ID" v-model="active_set_id" @keypress.enter="searchForSet"></v-text-field></v-list-item>
      <v-list-item link title="Random Set" @click="loadRandomSet"/>
      <!-- <v-list-item link title="Import from CSV" /> -->
    </v-list>
  </v-navigation-drawer>
  <v-main>
    <v-container class="justify-left">

        <!-- set details -->
        <v-card class="mb-3" color="indigo">
          <v-card-title>{{ set?.name }}</v-card-title>
          <v-card-text>
            <v-row>
              <v-col cols="12" sm="4"><v-img :src="set?.set_img_url"/></v-col>
              <v-col cols="12" sm="8">
                <v-list>
                  <v-list-item title="Set Number" :subtitle="set?.set_num" />
                  <v-list-item title="Year" :subtitle="set?.year" />
                  <v-list-item title="Number of Parts" :subtitle="set?.num_parts" />
                </v-list>
                <v-checkbox label="Show Picked" v-model="showPicked" />
              </v-col>
            </v-row>
            <v-row>
              <v-col>
                <v-progress-linear color="green-lighten-1" class="mt-3" :height="36" :model-value="Math.floor((kitted_parts / set?.num_parts) * 100)">
                  <template v-slot:default="{ value }">
                    {{ kitted_parts }} / {{ set?.num_parts }}
                  </template>
                </v-progress-linear>
              </v-col>
            </v-row>
          </v-card-text>
        </v-card>

        <hr class="mb-3"/>

        <!-- part list -->
        <v-row>
          <template v-for="part in parts">
            <v-col cols="12" sm="6" md="4" lg="3" v-if="showPicked || (kit[`${part.part.part_num}-${part.color.id}`] || 0) < part.quantity">
            <!-- standard card for non mobile -->
            <v-card 
              v-if="breakpoints.greaterOrEqual('sm').value"
              :color="(kit[`${part.part.part_num}-${part.color.id}`] || 0) < part.quantity ? '' : 'green-lighten-4'" 
              class="mb-3 justify-center">
              <v-card-title @click="highlightPart">{{ part.part.part_num }} ({{ part.color.name }})</v-card-title>
              <v-card-subtitle>{{ part.part.name }}</v-card-subtitle>
              <v-card-text>
                <v-container>
                  <v-row class="mb-3 justify-center">
                    <v-img :height="128" :src="part.part.part_img_url" class="border-medium border-medium rounded"/>
                  </v-row>
                  <v-row class="mb-3 justify-center">
                    <v-btn-group density="compact">
                      <v-btn color="red-lighten-3" @click="decrementPart(`${part.part.part_num}-${part.color.id}`)">
                        <v-icon icon="mdi-minus-thick"></v-icon>
                      </v-btn>
                      <v-btn color="green-lighten-2" @click="completePart(`${part.part.part_num}-${part.color.id}`)">
                        <v-icon>mdi-check-decagram</v-icon>
                      </v-btn>
                      <v-btn color="blue-lighten-2" @click="incrementPart(`${part.part.part_num}-${part.color.id}`)">
                        <v-icon icon="mdi-plus-thick" />
                      </v-btn>
                    </v-btn-group>
                  </v-row>
                  <v-row class="justify-center text-button">{{ kit[`${part.part.part_num}-${part.color.id}`] || 0 }} / {{ part.quantity }}</v-row>
                </v-container>
              </v-card-text>
            </v-card>

            <!-- xs card for mobile -->
            <v-card 
              v-else
              :color="(kit[`${part.part.part_num}-${part.color.id}`] || 0) < part.quantity ? '' : 'green-lighten-4'" 
              class="mb-3 justify-center">
              <v-card-text>
                <v-container>
                  <v-row class="mb-3 justify-center">
                    <v-col cols="3"><v-img :src="part.part.part_img_url" class="border-medium border-medium rounded"/></v-col>
                    <v-col cols="9">
                      <p class="text-h6">{{ part.part.part_num }} ({{ part.color.name }})</p>
                      <p class="font-weight-light">{{ part.part.name }}</p>
                    </v-col>
                  </v-row>
                  <v-row class="justify-center">
                    <v-col cols="3" class="text-center">{{ kit[`${part.part.part_num}-${part.color.id}`] || 0 }} / {{ part.quantity }}</v-col>
                    <v-col cols="9">
                      <v-btn-group density="compact">
                        <v-btn color="red-lighten-3" @click="decrementPart(`${part.part.part_num}-${part.color.id}`)">
                          <v-icon icon="mdi-minus-thick"></v-icon>
                        </v-btn>
                        <v-btn color="green-lighten-2" @click="completePart(`${part.part.part_num}-${part.color.id}`)">
                          <v-icon>mdi-check-decagram</v-icon>
                        </v-btn>
                        <v-btn color="blue-lighten-2" @click="incrementPart(`${part.part.part_num}-${part.color.id}`)">
                          <v-icon icon="mdi-plus-thick" />
                        </v-btn>
                      </v-btn-group>
                    </v-col>
                  </v-row>
                </v-container>
              </v-card-text>
            </v-card>
          </v-col>
          </template>
        </v-row>

        <hr class="mb-3"/>

        <!-- debug -->
        <v-card class="mb-3" color="red-lighten-3">
          <v-card-title @click="debug = !debug">Debug</v-card-title>
          <v-card-text v-if="debug">
            <pre>Rebricable API Key: {{ api_secret }}</pre>
            <hr class="my-3"/>
            <pre>{{ set }}</pre>
            <hr class="my-3"/>
            <pre>{{ parts }}</pre>
          </v-card-text>
        </v-card>

    </v-container>

  </v-main>
</v-layout>
</template>

