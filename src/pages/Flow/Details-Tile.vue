<script>
import CardTitle from '@/components/Card-Title'
import LabelEdit from '@/components/LabelEdit'
import Parameters from '@/components/Parameters'
import PrefectSchedule from '@/components/PrefectSchedule'
import { formatTime } from '@/mixins/formatTimeMixin'
import { parametersMixin } from '@/mixins/parametersMixin'
import { mapActions, mapGetters } from 'vuex'

export default {
  filters: {
    typeClass: val => val.split('.').pop()
  },
  components: {
    CardTitle,
    LabelEdit,
    Parameters,
    PrefectSchedule
  },
  mixins: [formatTime, parametersMixin],
  props: {
    flow: {
      type: Object,
      default: () => {}
    },
    flowGroup: {
      type: Object,
      default: () => {}
    },
    fullHeight: {
      required: false,
      type: Boolean,
      default: () => false
    },
    lastFlowRun: {
      type: Object,
      default: () => {}
    }
  },
  data() {
    return {
      paramInfoOpen: false,
      copiedText: {},
      tab: 'overview',
      //labels
      newLabel: '',
      labelMenuOpen: false,
      labelEditOpen: false,
      labelSearchInput: '',
      removingLabel: null,
      newLabels: null,
      disableRemove: false,
      disableAdd: false,
      valid: false,
      errorMessage: '',
      duplicateLabel: '',
      rules: {
        labelCheck: value => this.checkLabelInput(value) || this.errorMessage
      }
    }
  },
  computed: {
    ...mapGetters('tenant', ['role']),
    filteredStorage() {
      if (!this.flow.storage) return {}

      const filtered = ['__version__', 'prefect_version', 'flows']

      return Object.keys(this.flow.storage)
        .filter(key => !filtered.includes(key))
        .reduce((obj, key) => {
          obj[key] = this.flow.storage[key]
          return obj
        }, {})
    },
    flows() {
      if (!this.flow.storage || !this.flow.storage.flows) return null
      return this.flow.storage.flows
    },
    labels() {
      const labels =
        this.newLabels ||
        this.flowGroup?.labels ||
        this.flow?.environment?.labels ||
        []
      return labels?.slice().sort()
    },
    labelResetDisabled() {
      const labels = this.newLabels || this.labels
      return (
        Array.isArray(labels) &&
        Array.isArray(this.flow?.environment?.labels) &&
        labels.length === this.flow?.environment?.labels?.length &&
        labels.every(
          (val, index) => val === this.flow?.environment?.labels[index]
        )
      )
    },
    hasUser() {
      return this.flow?.created_by
    }
  },
  methods: {
    ...mapActions('alert', ['setAlert']),
    checkLabelInput(val) {
      const labels = this.newLabels || this.labels
      if (labels.includes(val) && !this.disableAdd) {
        this.errorMessage = 'Duplicate label'
        this.duplicateLabel = val
        this.valid = false
        return false
      }
      this.duplicateLabel = ''
      this.valid = true
      return true
    },
    removeLabel(labelToRemove) {
      this.removingLabel = labelToRemove
      this.disableRemove = true
      const labels = this.newLabels || this.labels
      const updatedArray = labels.filter(label => {
        return labelToRemove != label
      })
      this.editLabels(updatedArray)
    },
    addLabel() {
      if (!this.valid) return
      if (!this.newLabel) return
      this.disableAdd = true
      const labelArray = this.newLabels || this.labels.slice()
      labelArray.push(this.newLabel)
      this.editLabels(labelArray)
    },
    async editLabels(newLabels) {
      try {
        const { data } = await this.$apollo.mutate({
          mutation: require('@/graphql/Mutations/set-labels.gql'),
          variables: {
            flowGroupId: this.flowGroup.id,
            labelArray: newLabels
          }
        })
        if (data) {
          this.newLabels = newLabels || this.flow.environment.labels
          this.resetLabelSettings()
        } else {
          this.labelsError()
          this.resetLabelSettings()
        }
      } catch (e) {
        this.labelsError(e)
        this.resetLabelSettings()
      }
    },
    labelReset() {
      this.editLabels(null)
    },
    resetLabelSettings() {
      this.removingLabel = false
      this.disableAdd = false
      this.newLabel = ''
      this.disableRemove = false
    },
    labelsError(e) {
      const message = e
        ? `There was a problem: ${e}`
        : 'There was a problem updating your labels.  Please try again.'
      this.setAlert({
        alertShow: true,
        alertMessage: message,
        alertType: 'error'
      })
    },
    clickAndCopyable(field) {
      return ['image_tag', 'image_name', 'registry_url'].includes(field)
    },
    copyToClipboard(value) {
      this.copiedText = {}
      this.copiedText[value] = true
      navigator.clipboard.writeText(value)

      setTimeout(() => {
        this.copiedText = {}
        this.copiedText[value] = false
      }, 600)
    }
  }
}
</script>

<template>
  <v-card
    class="pa-2 pr-0"
    tile
    :style="{
      height: fullHeight ? '100%' : 'auto',
      'max-height': fullHeight ? '100%' : 'auto'
    }"
  >
    <v-system-bar
      :color="flow.archived ? 'grey' : lastFlowRun ? lastFlowRun.state : 'grey'"
      :height="5"
      absolute
    >
      <!-- We should include a state icon here when we've got those -->
      <!-- <v-icon>{{ flow.flow_runs[0].state }}</v-icon> -->
    </v-system-bar>
    <CardTitle :icon="flow.archived ? 'archive' : 'pi-flow'">
      <v-row slot="title" no-gutters class="d-flex align-center">
        <v-col cols="8">
          <div class="text-truncate pb-1">
            {{ flow.name }}
          </div>
          <div
            class="subtitle-2 grey--text text--darken-2 caption position-absolute font-weight-medium"
            style="bottom: 2px;"
          >
            {{ `Version ${flow.version}` }}
          </div>
        </v-col>
      </v-row>

      <div slot="action" class="d-flex flex-column align-end">
        <v-btn
          depressed
          small
          tile
          icon
          class="button-transition w-100 d-flex justify-end"
          :color="tab == 'overview' ? 'primary' : ''"
          :style="{
            'border-right': `3px solid ${
              tab == 'overview' ? 'var(--v-primary-base)' : 'transparent'
            }`,
            'box-sizing': 'content-box',
            'min-width': '100px'
          }"
          @click="tab = 'overview'"
        >
          Overview
          <v-icon small>calendar_view_day</v-icon>
        </v-btn>

        <v-btn
          depressed
          small
          tile
          icon
          class="button-transition w-100 d-flex justify-end"
          :color="tab == 'details' ? 'primary' : ''"
          :style="{
            'border-right': `3px solid ${
              tab == 'details' ? 'var(--v-primary-base)' : 'transparent'
            }`,
            'box-sizing': 'content-box',
            'min-width': '100px'
          }"
          @click="tab = 'details'"
        >
          Details
          <v-icon small>notes</v-icon>
        </v-btn>
      </div>
    </CardTitle>

    <v-card-text class="pl-12 pt-2 card-content">
      <v-fade-transition hide-on-leave>
        <div v-if="tab == 'overview'">
          <v-list-item dense class="px-0">
            <v-list-item-content>
              <v-list-item-subtitle class="caption">
                Created <span v-if="hasUser">by</span>
              </v-list-item-subtitle>
              <div v-if="hasUser" class="subtitle-2">
                {{ flow.created_by.username }}
              </div>
              <div class="caption" :class="{ 'font-weight-bold': !hasUser }">
                {{ formatTime(flow.created) }}
              </div>
            </v-list-item-content>
          </v-list-item>

          <v-list-item v-if="flow.core_version" dense class="px-0">
            <v-list-item-content>
              <v-list-item-subtitle class="caption">
                Prefect Core Version:
              </v-list-item-subtitle>
              <div class="subtitle-2">{{ flow.core_version }}</div>
            </v-list-item-content>
          </v-list-item>

          <v-list-item dense class="px-0">
            <v-list-item-content>
              <v-list-item-subtitle class="caption">
                Schedule
              </v-list-item-subtitle>
              <div class="subtitle-2">
                <PrefectSchedule
                  v-if="flow.schedule"
                  :schedule="flow.schedule"
                />
                <span v-else>None</span>
              </div>
            </v-list-item-content>
          </v-list-item>

          <LabelEdit :flow="flow" :flow-group="flowGroup" />
        </div>
      </v-fade-transition>

      <v-fade-transition hide-on-leave>
        <div v-if="tab == 'details'">
          <v-list-item dense class="px-0">
            <v-list-item-content>
              <v-list-item-subtitle class="grey--text text--darken-3">
                General
              </v-list-item-subtitle>
              <v-divider style="max-width: 50%;" />
              <v-list-item-subtitle class="caption">
                <v-row no-gutters>
                  <v-col cols="6">
                    Flow ID
                  </v-col>
                  <v-col
                    cols="6"
                    class="text-right font-weight-bold text-truncate"
                  >
                    <v-tooltip bottom>
                      <template v-slot:activator="{ on }">
                        <span
                          class="cursor-pointer show-icon-hover-focus-only pa-2px"
                          role="button"
                          @click="copyToClipboard(flow.id)"
                          v-on="on"
                        >
                          <v-icon x-small class="mb-2px mr-2" tabindex="0">
                            {{ copiedText[flow.id] ? 'check' : 'file_copy' }}
                          </v-icon>
                          {{ flow.id }}
                        </span>
                      </template>
                      <span>
                        {{ flow.id }}
                      </span>
                    </v-tooltip>
                  </v-col>
                </v-row>
              </v-list-item-subtitle>
              <v-list-item-subtitle class="caption">
                <v-row no-gutters>
                  <v-col cols="6">
                    Flow Group ID
                  </v-col>
                  <v-col
                    cols="6"
                    class="text-right font-weight-bold text-truncate"
                  >
                    <v-tooltip bottom>
                      <template v-slot:activator="{ on }">
                        <span
                          class="cursor-pointer show-icon-hover-focus-only pa-2px"
                          role="button"
                          @click="copyToClipboard(flow.flow_group_id)"
                          v-on="on"
                        >
                          <v-icon x-small class="mb-2px mr-2" tabindex="0">
                            {{
                              copiedText[flow.flow_group_id]
                                ? 'check'
                                : 'file_copy'
                            }}
                          </v-icon>
                          {{ flow.flow_group_id }}
                        </span>
                      </template>
                      <span>
                        {{ flow.flow_group_id }}
                      </span>
                    </v-tooltip>
                  </v-col>
                </v-row>
              </v-list-item-subtitle>
              <v-list-item-subtitle class="caption">
                <v-row no-gutters>
                  <v-col cols="6">
                    Version Group ID
                  </v-col>
                  <v-col
                    cols="6"
                    class="text-right font-weight-bold text-truncate"
                  >
                    <v-tooltip bottom>
                      <template v-slot:activator="{ on }">
                        <span
                          class="cursor-pointer show-icon-hover-focus-only pa-2px"
                          role="button"
                          @click="copyToClipboard(flow.version_group_id)"
                          v-on="on"
                        >
                          <v-icon x-small class="mb-2px mr-2" tabindex="0">
                            {{
                              copiedText[flow.version_group_id]
                                ? 'check'
                                : 'file_copy'
                            }}
                          </v-icon>
                          {{ flow.version_group_id }}
                        </span>
                      </template>
                      <span>
                        {{ flow.version_group_id }}
                      </span>
                    </v-tooltip>
                  </v-col>
                </v-row>
              </v-list-item-subtitle>
            </v-list-item-content>
          </v-list-item>

          <v-list-item dense class="px-0">
            <v-list-item-content>
              <v-list-item-subtitle class="grey--text text--darken-3">
                Environment
              </v-list-item-subtitle>
              <v-divider style="max-width: 50%;" />
              <v-list-item-subtitle class="caption">
                <v-row v-if="flow.environment.type" no-gutters>
                  <v-col cols="6">
                    Type
                  </v-col>
                  <v-col cols="6" class="text-right font-weight-bold">
                    {{ flow.environment.type }}
                  </v-col>
                </v-row>

                <v-row v-if="flow.environment.executor" no-gutters>
                  <v-col cols="6">
                    Executor
                  </v-col>
                  <v-col cols="6" class="text-right font-weight-bold">
                    {{ flow.environment.executor | typeClass }}
                  </v-col>
                </v-row>
              </v-list-item-subtitle>
            </v-list-item-content>
          </v-list-item>

          <v-list-item v-if="flow.storage" dense class="px-0 mt-2">
            <v-list-item-content>
              <v-list-item-subtitle class="grey--text text--darken-3">
                Storage
              </v-list-item-subtitle>
              <v-divider style="max-width: 50%;" />
              <v-list-item-subtitle class="caption">
                <v-row
                  v-for="(value, key) in filteredStorage"
                  :key="key"
                  no-gutters
                >
                  <v-col cols="4">
                    {{ key }}
                  </v-col>
                  <v-col
                    cols="8"
                    class="text-right font-weight-bold text-truncate"
                  >
                    <v-tooltip
                      v-if="clickAndCopyable(key)"
                      bottom
                      max-width="300"
                    >
                      <template v-slot:activator="{ on }">
                        <span
                          class="cursor-pointer show-icon-hover-focus-only pa-2px"
                          :class="{
                            'bg-gray-transition': copiedText[value],
                            'bg-white-transition': !copiedText[value]
                          }"
                          role="button"
                          @click="copyToClipboard(value)"
                          v-on="on"
                        >
                          <v-icon x-small class="mb-2px mr-2" tabindex="0">{{
                            copiedText[value] ? 'check' : 'file_copy'
                          }}</v-icon
                          >{{ value }}
                        </span>
                      </template>
                      <span>
                        {{ value }}
                      </span>
                    </v-tooltip>
                    <span v-else>
                      {{ value }}
                    </span>
                  </v-col>
                </v-row>
              </v-list-item-subtitle>
            </v-list-item-content>
          </v-list-item>

          <v-list-item v-if="flows" dense class="px-0 mt-2">
            <v-list-item-content>
              <v-list-item-subtitle class="grey--text text--darken-3">
                Flow Locations
              </v-list-item-subtitle>
              <v-divider style="max-width: 50%;" />
              <v-list-item-subtitle class="caption">
                <v-row
                  v-for="(value, key) in flows"
                  :key="key"
                  no-gutters
                  class="my-1"
                >
                  <v-col cols="12">
                    {{ key }}
                  </v-col>
                  <v-col cols="12" class="font-weight-bold">
                    {{ value }}
                  </v-col>
                </v-row>
              </v-list-item-subtitle>
            </v-list-item-content>
          </v-list-item>

          <v-list-item
            v-if="flow.parameters && flow.parameters.length > 0"
            dense
            two-line
            class="px-0"
          >
            <v-list-item-content>
              <v-list-item-subtitle class="grey--text text--darken-3">
                Parameters
                <v-menu
                  v-model="paramInfoOpen"
                  :close-on-content-click="false"
                  offset-y
                  open-on-hover
                >
                  <template v-slot:activator="{ on }">
                    <v-btn text icon x-small v-on="on">
                      <v-icon>
                        info
                      </v-icon>
                    </v-btn>
                  </template>
                  <v-card tile class="pa-0" max-width="220">
                    <v-card-text class="pb-0">
                      <p>
                        Here you can see the default paramaters for your
                        flow.</p
                      ><p>
                        If you want to update your flow group's parameters, you
                        can do so on the parameters tab in
                        <router-link
                          :to="{ name: 'flow', query: { tab: 'settings' } }"
                          @click.native="paramInfoOpen = false"
                          >Flow Settings</router-link
                        >.
                      </p>
                      <p>
                        Refer to the
                        <a
                          href="https://docs.prefect.io/core/concepts/parameters.html#parameters"
                          target="_blank"
                          rel="noopener noreferrer"
                          @click="paramInfoOpen = false"
                        >
                          documentation</a
                        >
                        <sup
                          ><v-icon x-small>
                            open_in_new
                          </v-icon></sup
                        >
                        for more details on parameters.
                      </p>
                    </v-card-text>
                    <v-card-actions class="pt-0">
                      <v-spacer></v-spacer>
                      <v-btn small text @click="paramInfoOpen = false"
                        >Close</v-btn
                      >
                    </v-card-actions>
                  </v-card>
                </v-menu>
              </v-list-item-subtitle>
              <v-divider style="max-width: 50%;" />
              <v-list-item-subtitle>
                <Parameters :parameters="defaultParameters"></Parameters>
              </v-list-item-subtitle>
            </v-list-item-content>
          </v-list-item>
        </div>
      </v-fade-transition>
    </v-card-text>
  </v-card>
</template>

<style lang="scss">
.bg-gray-transition {
  background-color: #efefef;
  transition: background-color 300ms;
}

.bg-white-transition {
  background-color: #fff;
  transition: background-color 300ms;
}

.card-content {
  max-height: 254px;
  overflow-y: scroll;
}

.cursor-pointer {
  cursor: pointer;
}

.label-search {
  border-radius: 0 !important;
  font-size: 0.85rem !important;

  .v-icon {
    font-size: 20px !important;
  }
}

.max-h-300 {
  max-height: 300px;
}

.mb-2px {
  margin-bottom: 2px;
}

.pa-2px {
  padding: 2px;
}

.show-icon-hover-focus-only {
  .v-icon {
    visibility: hidden;
  }

  &:hover,
  &:focus {
    .v-icon {
      visibility: visible;
    }
  }
}
/* stylelint-disable */

.v-list-item__action--stack {
  align-items: flex-start;
  flex-direction: row;
}

.v-text-field input {
  width: 100px;
}

.v-list-item__content {
  overflow: scroll;
}

.w-100 {
  width: 100% !important;
}
</style>
