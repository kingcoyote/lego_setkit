<script lang="ts" setup>

import { useLocalStorage } from "@vueuse/core"

const set = useLocalStorage('setkit.set', {});
const parts = useLocalStorage('setkit.parts', [])
const kit = useLocalStorage('setkit.kit', {});

const set_list = [
  "60378",
  "30161",
  "76177",
  "41737",
  "31138",
  "71363",
  "60383",
  "10786",
  "10788",
  "77092",
  "40721",
  "21330",
  "40599",
  "77049",
  "77046",
  "77048",
  "40429",
  "40346",
  "21047",
  "43213",
  "71454",
  "31120",
  "10696",
  "21309",
  "40519",
  "31147",
  "31134",
  "21251",
  "10790",
  "21166",
  "21241",
  "21546",
  "21242",
  "21179",
  "21170",
  "21152",
  "21189",
  "21240",
]

const api_secret = useRuntimeConfig().public.rebrickableApiSecret;

async function searchForSet() {
  const set_id = set_list[Math.floor(Math.random() * set_list.length)]
  set.value = {}
  parts.value = []
  kit.value = {}

  let response = await fetch(`https://rebrickable.com/api/v3/lego/sets/${set_id}-1/?key=${api_secret}`);
  set.value = await response.json()

  let next = `https://rebrickable.com/api/v3/lego/sets/${set_id}-1/parts?key=${api_secret}`

  while (next != null) {
    response = await fetch(next);
    let result = await response.json();
    parts.value = parts.value.concat(result['results']);
    next = result['next'];
  }
}

function incrementPart(part_num) {
  if (kit.value[part_num] === undefined) {
    kit.value[part_num] = 0;
  }

  kit.value[part_num] += 1;
}

function decrementPart(part_num) {
  if (kit.value[part_num] === undefined) {
    kit.value[part_num] = 0;
  }

  kit.value[part_num] -= 1;

  if (kit.value[part_num] < 0) {
    kit.value[part_num] = 0;
  }
}

const debug=ref(false)

</script>

<template>
  <v-layout class="rounded rounded-md">
    <v-app-bar>
      <v-app-bar-title>Lego Setkit</v-app-bar-title>
    </v-app-bar>
    <v-navigation-drawer :v-model="true">
      <v-list>
        <v-list-item link title="Search for Set" @click="searchForSet"/>
        <v-list-item link title="Import from CSV" />
      </v-list>
    </v-navigation-drawer>
    <v-main>
      <v-container class="ga-3">

          <v-card class="mb-3" color="indigo">
            <v-card-title>{{ set.name }}</v-card-title>
            <v-card-text>
              <v-row>
                <v-col md="4"><v-img :width="300" aspect-ratio="1/1" :src="set.set_img_url"/></v-col>
                <v-col md="8">
                  <v-list>
                    <v-list-item title="Set Number" :subtitle="set.set_num" />
                    <v-list-item title="Year" :subtitle="set.year" />
                    <v-list-item title="Num. Parts" :subtitle="set.num_parts" />
                  </v-list>
                </v-col>
              </v-row>
            </v-card-text>
          </v-card>

          <hr class="mb-3"/>

          <div v-for="part in parts" >
            <v-card v-if="(kit[part.part.part_num] || 0) < part.quantity" class="mb-3">
              <v-card-title>{{ part.quantity}}x {{ part.part.name }}</v-card-title>
              <v-card-text>
                
                <v-row>
                  
                  <v-col sm="2"><v-img :height="48" aspect-ratio="1/1" :src="part.part.part_img_url"/></v-col>
                  <v-col sm="10">
                    <v-row>
                      <v-col>
                        <v-btn class="h-100" @click="decrementPart(part.part.part_num)">-</v-btn>
                        <v-btn @click="incrementPart(part.part.part_num)">+</v-btn>
                      </v-col>
                      <v-col>{{ part.part.part_num }}</v-col>
                      <v-col>{{ kit[part.part.part_num] || 0 }} / {{ part.quantity }}</v-col>
                      
                      
                      <v-col>
                        
                      </v-col>
                    </v-row>
                    
                  </v-col>
                </v-row>
              </v-card-text>
            </v-card>
          </div>

          <hr class="mb-3"/>

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

