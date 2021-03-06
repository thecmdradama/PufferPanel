<template>
  <v-row>
    <v-col cols="12">
      <v-card>
        <v-card-title>
          <span v-text="$t(create ? 'templates.New' : 'templates.Edit')" />
          <div class="flex-grow-1" />
          <v-btn-toggle
            v-model="currentMode"
            borderless
            dense
            mandatory
          >
            <v-btn
              value="editor"
              @click="updateEditor()"
              v-text="$t('templates.Editor')"
            />
            <v-btn
              value="json"
              @click="updateJson()"
              v-text="$t('templates.Json')"
            />
          </v-btn-toggle>
        </v-card-title>
        <v-card-text>
          <v-row>
            <v-col>
              <ui-input
                v-model="name"
                :label="$t('common.Name')"
                :disabled="!create"
                hide-details
              />
            </v-col>
          </v-row>
          <v-row v-if="currentMode === 'editor'">
            <v-col
              cols="12"
              md="6"
            >
              <ui-input
                v-model="templateObj.display"
                :label="$t('templates.Display')"
                hide-details
              />
            </v-col>
            <v-col
              cols="12"
              md="6"
            >
              <ui-input
                v-model="templateObj.type"
                :label="$t('templates.Type')"
                hide-details
              />
            </v-col>
          </v-row>
          <v-row v-if="loading">
            <v-col cols="5" />
            <v-col cols="2">
              <v-progress-circular
                indeterminate
                class="mr-2"
              />
              <strong v-text="$t('common.Loading')" />
            </v-col>
          </v-row>
        </v-card-text>
      </v-card>
    </v-col>
    <v-col
      v-if="currentMode === 'json' && !loading"
      cols="12"
    >
      <v-card>
        <v-card-text>
          <v-row>
            <v-col>
              <ace
                ref="editor"
                v-model="template"
                :editor-id="name + 'json'"
                :theme="isDark() ? 'monokai' : 'github'"
                height="50vh"
                lang="json"
                @editorready="$refs.editor.setValue(template)"
              />
            </v-col>
          </v-row>
        </v-card-text>
      </v-card>
    </v-col>
    <v-col
      v-if="currentMode === 'editor' && !loading"
      cols="12"
    >
      <v-card>
        <v-card-title v-text="$t('templates.Variables')" />
        <v-card-text class="pb-1">
          <template-variables v-model="templateObj.data" />
        </v-card-text>
      </v-card>
    </v-col>
    <v-col
      v-if="currentMode === 'editor' && !loading"
      cols="12"
    >
      <v-card>
        <v-card-title v-text="$t('templates.Install')" />
        <v-card-text class="pb-1">
          <template-processors
            v-model="templateObj.install"
            name="install"
          />
        </v-card-text>
      </v-card>
    </v-col>
    <v-col
      v-if="currentMode === 'editor' && !loading"
      cols="12"
    >
      <v-card>
        <v-card-title v-text="$t('templates.RunConfig')" />
        <v-card-text class="pb-1">
          <template-run v-model="templateObj.run" />
        </v-card-text>
      </v-card>
    </v-col>
    <v-col
      v-if="currentMode === 'editor' && !loading"
      cols="12"
    >
      <v-card>
        <v-card-title v-text="$t('templates.Shutdown')" />
        <v-card-text class="pb-1">
          <template-shutdown v-model="templateObj.run" />
        </v-card-text>
      </v-card>
    </v-col>
    <v-col
      v-if="currentMode === 'editor' && !loading"
      cols="12"
    >
      <v-card>
        <v-card-title v-text="$t('templates.PreHook')" />
        <v-card-text class="pb-1">
          <template-processors
            v-model="templateObj.run.pre"
            name="pre"
          />
        </v-card-text>
      </v-card>
    </v-col>
    <v-col
      v-if="currentMode === 'editor' && !loading"
      cols="12"
    >
      <v-card>
        <v-card-title v-text="$t('templates.PostHook')" />
        <v-card-text class="pb-1">
          <template-processors
            v-model="templateObj.run.post"
            name="post"
          />
        </v-card-text>
      </v-card>
    </v-col>
    <v-col
      v-if="currentMode === 'editor' && !loading"
      cols="12"
    >
      <v-card>
        <v-card-title v-text="$t('templates.EnvVars')" />
        <v-card-text class="pb-1">
          <ui-map-input
            v-model="templateObj.run.environmentVars"
            @input="$forceUpdate()"
          />
        </v-card-text>
      </v-card>
    </v-col>
    <v-col
      v-if="currentMode === 'editor' && !loading"
      cols="12"
    >
      <v-card>
        <v-card-title v-text="$t('templates.Environments')" />
        <v-card-text>
          <template-environments v-model="templateObj.supportedEnvironments" />
          <ui-select
            v-if="Object.keys(templateObj.supportedEnvironments).length > 0"
            v-model="templateObj.environment"
            :label="$t('templates.DefaultEnvironment')"
            :items="configuredEnvironments"
            class="pt-4"
          />
        </v-card-text>
      </v-card>
    </v-col>
    <v-col cols="12">
      <v-btn
        color="success"
        large
        block
        @click="save"
        v-text="$t('common.Save')"
      />
    </v-col>
    <v-col cols="12">
      <v-btn
        v-if="!create"
        color="error"
        block
        @click="remove()"
        v-text="$t('common.Delete')"
      />
    </v-col>
  </v-row>
</template>

<script>
import { isDark } from '@/utils/dark'

export default {
  data () {
    return {
      currentMode: 'editor',
      loading: false,
      create: this.$route.params.id === undefined,
      name: this.$route.params.id === undefined ? '' : this.$route.params.id,
      stopType: 'command',
      template: '',
      templateObj: this.withDefaults({})
    }
  },
  computed: {
    configuredEnvironments () {
      return this.templateObj.supportedEnvironments.map(elem => { return { text: elem.type, value: elem } })
    }
  },
  watch: {
    'templateObj.supportedEnvironments' (val) {
      if (this.templateObj.environment) {
        this.templateObj.environment = val.filter(elem => {
          return elem.type === this.templateObj.environment.type
        })[0]
      }
    }
  },
  mounted () {
    if (!this.create) this.loadData()
  },
  methods: {
    withDefaults (obj) {
      const fixType = element => {
        if (element.type === 'download' && typeof element.files === 'string') element.files = [element.files]
        if (element.type === 'command' && typeof element.commands === 'string') element.commands = [element.commands]
        return element
      }

      if (!obj.data) obj.data = {}
      Object.keys(obj.data).forEach(key => {
        if (!obj.data[key].type) obj.data[key].type = 'string'
      })
      if (!obj.run) obj.run = {}
      if (!obj.run.environmentVars) obj.run.environmentVars = {}
      if (!obj.run.pre) obj.run.pre = []
      if (!obj.run.post) obj.run.post = []
      if (!obj.supportedEnvironments) obj.supportedEnvironments = []
      if (!obj.install) obj.install = []
      obj.install.map(fixType)
      obj.run.pre.map(fixType)
      obj.run.post.map(fixType)
      return obj
    },
    async loadData () {
      this.loading = true
      const template = await this.$api.getTemplate(this.$route.params.id)
      delete template.readme
      this.template = JSON.stringify(template, undefined, 4)
      this.templateObj = this.withDefaults(template)
      if (this.$refs.editor && this.$refs.editor.ready) this.$refs.editor.setValue(this.template)
      this.loading = false
    },
    async remove () {
      if (this.create) return
      await this.$api.deleteTemplate(this.name)
      this.$toast.success(this.$t('templates.Deleted'))
      this.$router.push({ name: 'Templates' })
    },
    async save () {
      if (this.currentMode === 'editor') this.updateJson()
      await this.$api.createTemplate(this.name, this.template)
      this.$toast.success(this.$t('templates.SaveSuccess'))
      if (this.create) this.$router.push({ name: 'Template', params: { id: this.name } })
    },
    updateEditor () {
      this.templateObj = JSON.parse(this.template)
    },
    updateJson () {
      this.templateObj.name = this.name
      this.templateObj.run.stopCode = this.templateObj.run.stopCode * 1
      this.template = JSON.stringify(this.templateObj, undefined, 4)
      if (this.$refs.editor && this.$refs.editor.ready) this.$refs.editor.setValue(this.template)
    },
    isDark
  }
}
</script>
