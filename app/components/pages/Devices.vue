<template>
  <div>
    <help :visible="help" @close="help = false"></help>

    <h1 class="title">
      Périphériques
      <a href="#" @click.prevent="help = true" data-balloon="Aide" data-balloon-pos="up">
        <span class="icon is-medium">
          <i class="fa fa-question-circle-o"></i>
        </span>
      </a>
    </h1>
    <h2 class="subtitle">
      Tous les périphériques de votre maison
    </h2>

    <nav class="level">
      <div class="level-left">
        <div class="level-item">
          <p class="control has-addons">
            <input v-model="tagInput.value" @focus="tagInput.focus = true" @blur="tagInput.focus = false" class="input" type="text" placeholder="Filtrer ou créer un tag">
            <ul v-if="tagInput.focus == true" id="autocomplete-dropdown">
              <li v-if="tagInput.value !== ''"><a href="" @click.prevent @mousedown.prevent="createTag(tagInput.value)"><span class="icon is-small"><i class="fa fa-plus"></i></span> Créer le tag <b>{{ tagInput.value }}</b></a></li>
              <div v-for="tag in dropdownTags">
              <li v-if="!tag.id.includes('room:')"><a href="" @click.prevent @mousedown.prevent="addTag(tag)"><span class="tag"><span class="icon is-small"><i class="fa fa-tag"></i></span>&nbsp;{{ tag.id }}<span :data-balloon="canDeleteTag(tag.id) ? 'Supprimer le tag' : 'Impossible de supprimer ce tag car il est encore affecté'" data-balloon-pos="right"><button @mousedown.prevent.stop="deleteTag(tag.id)" class="delete is-small" :disabled="!canDeleteTag(tag.id)"></button></span></span></a></li>
              </div>
            </ul>
          </p>
        </div>
        <div class="level-item">
          <span v-for="id in selectedTagsIds" class="tag">
            <div v-for="floor in infrastructure.house.floors">
                  <div v-for="room in floor.rooms">
                    <span v-if="id === room.tagId">
                    <span class="icon is-small">
                    <i class="fa fa-tag"></i></span>&nbsp;{{ room.name }}<button @click="removeCurrentTag(room.tagId)" class="delete is-small"></button></span>
                  </div>
            </div>
             <span v-if="!id.includes('room:')">
            <span class="icon is-small">
             <i  class="fa fa-tag"></i></span>&nbsp;{{ infrastructure.tags[id].id }}<button @click="removeCurrentTag(id)" class="delete is-small"></button></span>
          </span>
        </div>
      </div>

      <div class="level-item has-text-centered">
        <p class="subtitle is-5">
          <strong>{{ filteredNodes.length }}</strong> périphériques correspondants
        </p>
      </div>

      <div class="level-right">
        <p class="level-item">
          <strong v-if="selectedDeviceState === DEVICE_STATES.ONLINE">En ligne</strong>
          <a v-else @click="selectedDeviceState = DEVICE_STATES.ONLINE" class="button is-success">En ligne</a>
        </p>
        <p class="level-item">
          <strong v-if="selectedDeviceState === null">Tous</strong>
          <a v-else @click="selectedDeviceState = null" class="button">Tous</a>
        </p>
        <p class="level-item">
          <strong v-if="selectedDeviceState === DEVICE_STATES.OFFLINE">Hors ligne</strong>
          <a v-else @click="selectedDeviceState = DEVICE_STATES.OFFLINE" class="button is-danger">Hors ligne</a>
        </p>
      </div>
    </nav>

    <div class="columns is-multiline is-mobile">
      <component v-for="node in filteredNodes" :is="types[node.type]" :nodeData="node"></component>
    </div>
  </div>
</template>

<script>
import {mapState, mapActions} from 'eva.js'

import Help from '../help/Devices'

import SwitchDevice from '../devices/Switch'
import LightDevice from '../devices/Light'
import TemperatureDevice from '../devices/Temperature'
import HumidityDevice from '../devices/Humidity'
import ShuttersDevice from '../devices/Shutters'
import DoorDevice from '../devices/Door'
import WindowDevice from '../devices/Window'
import LockDevice from '../devices/Lock'
import HeaterDevice from '../devices/Heater'
import SoundDevice from '../devices/Sound'
import LuminosityDevice from '../devices/Luminosity'
import MotionDevice from '../devices/Motion'
import BuzzerDevice from '../devices/Buzzer'
import ButtonDevice from '../devices/Button'

export default {
  data () {
    return {
      help: false,
      types: {
        'temperature': TemperatureDevice,
        'light': LightDevice,
        'switch': SwitchDevice,
        'humidity': HumidityDevice,
        'shutters': ShuttersDevice,
        'door': DoorDevice,
        'window': WindowDevice,
        'lock': LockDevice,
        'heater': HeaterDevice,
        'sound': SoundDevice,
        'luminosity': LuminosityDevice,
        'motion': MotionDevice,
        'buzzer': BuzzerDevice,
        'button': ButtonDevice
      },
      tagInput: { value: '', focus: false },
      selectedTagsIds: [],
      DEVICE_STATES: {
        ONLINE: 'ONLINE',
        OFFLINE: 'OFFLINE'
      },
      selectedDeviceState: null
    }
  },
  components: { Help },
  mounted () {
    if (this.route.query.tag !== undefined) {
      this.selectedTagsIds.push(this.route.query.tag)
    }
  },
  computed: {
    dropdownTags () {
      return Object.values(this.infrastructure.tags).filter((tag) => {
        return tag.id.includes(this.tagInput.value) && !this.selectedTagsIds.includes(tag.id)
      })
    },
    filteredNodes () {
      const nodes = []
      Object.values(this.infrastructure.devices).forEach((device) => {
        if (this.selectedDeviceState) {
          const onlineWanted = this.selectedDeviceState === this.DEVICE_STATES.ONLINE
          if (device.online !== onlineWanted) return
        }

        // device is ok

        Object.values(device.nodes).forEach((node) => {
          for (const tagId of this.selectedTagsIds) {
            if (!node.tags.includes(tagId)) return
          }

          // node is ok

          nodes.push({ device, ...node })
        })
      })

      return nodes
    },
    ...mapState(['infrastructure', 'route'])
  },
  methods: {
    addTag (tag) {
      this.selectedTagsIds.push(tag.id)
    },
    addTagRoom (room) {
      this.selectedTagsIds.push(room.tagId)
    },
    removeCurrentTag (tagId) {
      this.selectedTagsIds.splice(this.selectedTagsIds.indexOf(tagId), 1)
    },
    async createTag (tagId) {
      await this.createTagAction({ id: tagId })
    },
    canDeleteTag (tagId) {
      for (let device of Object.values(this.infrastructure.devices)) {
        for (let node of Object.values(device.nodes)) {
          if (node.tags.includes(tagId)) return false
        }
      }

      return true
    },
    async deleteTag (tagId) {
      await this.deleteTagAction({ tagId: tagId })
    },
    ...mapActions({ createTagAction: 'createTag', deleteTagAction: 'deleteTag' })
  }
}

</script>

<style lang="sass" scoped>
  ul#autocomplete-dropdown
    position: absolute
    top: 31px

    width: 166px

    padding: 10px

    background-color: #fff
    border: 1px solid #dbdbdb
    border-radius: 0 0 3px 3px

    z-index: 1

    li
      margin-bottom: 5px

      &:last-of-type
        margin-bottom: 0
</style>
