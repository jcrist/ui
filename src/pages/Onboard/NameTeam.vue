<script>
import { mapActions, mapGetters } from 'vuex'

import { teamProfileMixin } from '@/mixins/teamProfileMixin.js'

export default {
  mixins: [teamProfileMixin],
  data() {
    return {
      // Form's computed height
      height: '0px',

      loading: 0,

      // Reveal animation bools
      revealNote: false,
      revealNameInput: false,
      revealUrlInput: false,
      revealConfirm: false
    }
  },
  computed: {
    ...mapGetters('user', ['user']),
    disabled() {
      return this.loading > 0 || !this.revealConfirm
    }
  },
  mounted() {
    this.tenantChanges.name = this.tenant.name
    this.tenantChanges.slug = this.tenant.slug

    setTimeout(() => {
      this.revealNote = true

      setTimeout(() => {
        this.height = getComputedStyle(this.$refs['main-row']).height
      })
    }, 500)

    setTimeout(() => {
      this.revealNameInput = true
      this.revealUrlInput = true
      this.revealConfirm = true

      setTimeout(() => {
        this.height = getComputedStyle(this.$refs['main-row']).height
      })
    }, 1000)
  },
  methods: {
    ...mapActions('tenant', ['getTenants', 'updateTenantSettings']),
    async createLicense() {
      this.loading++

      try {
        // Create the stripe customer (necessary for creating a self serve license)
        await this.$apollo.mutate({
          mutation: require('@/graphql/License/update-stripe-customer.gql'),
          variables: {
            input: {
              email: this.user.email,
              name: `${this.user.first_name}${this.user.last_name}`.trim(),
              source: null
            }
          }
        })

        // Create the self serve license
        await this.$apollo.mutate({
          mutation: require('@/graphql/License/create-self-serve-license.gql'),
          variables: {
            input: {
              confirm: true,
              users: 1,
              stripe_coupon_id: null
            }
          }
        })
      } catch (e) {
        if (!e?.message.includes('This tenant already has an active license'))
          this.updateServerError = true
      }

      this.loading--
      return
    },
    async updateTenant() {
      if (this.nameErrors.length > 0 || this.slugErrors.length > 0) {
        return
      }

      this.updateServerError = false
      this.loading++

      // Async tenant checks
      await this.checkNameAsync()
      await this.checkSlugAsync()

      if (
        this.tenantChanges.slug == this.tenant.slug &&
        this.tenantChanges.name == this.tenant.name
      ) {
        await this.updateTenantSettings({
          teamNamed: true
        })
      } else {
        try {
          await this.$apollo.mutate({
            mutation: require('@/graphql/Tenant/update-tenant.gql'),
            variables: {
              name: this.tenantChanges.name || this.tenant.name,
              slug: this.tenantChanges.slug || this.tenant.slug
            }
          })

          await this.updateTenantSettings({
            teamNamed: true
          })

          await this.getTenants()
          await this.setCurrentTenant(
            this.tenantChanges.slug || this.tenant.slug
          )
        } catch (e) {
          this.updateServerError = true
        }
      }

      this.loading--
      await this.createLicense()
      if (!this.updateServerError) this.goToResources()
    },
    goToResources() {
      this.revealNameInput = false
      this.revealUrlInput = false
      this.revealConfirm = false

      this.$router.push({
        name: 'onboard-resources',
        params: { tenant: this.tenant.slug }
      })
    }
  },
  apollo: {}
}
</script>

<template>
  <v-card
    v-if="tenant.id"
    class="text-center mx-auto px-12 py-8 white--text"
    flat
    tile
    style="width: fit-content !important;"
    color="transparent"
  >
    <v-row
      align="center"
      justify="center"
      :style="{ 'max-height': height }"
      class="transition-height"
    >
      <div ref="main-row">
        <transition-group name="fade">
          <v-col v-if="revealNote" key="name" cols="12" class="pb-0">
            <div class="display-1 text-center">
              Let's start by creating your team
            </div>
          </v-col>

          <v-col v-if="revealNote" key="revealNote" cols="12">
            <div class="body-2 text--darken-1">
              (You can always change this later)
            </div>
          </v-col>

          <v-col
            v-if="revealNameInput"
            key="input-1"
            cols="12"
            class="my-2 mt-12 name-team-input mx-auto"
          >
            <div class="overline">
              Team Name
            </div>
            <div v-if="tenant.role !== 'TENANT_ADMIN'" class="headline">
              {{ tenant.name }}
            </div>
            <v-text-field
              v-if="tenant.role == 'TENANT_ADMIN'"
              v-model="name"
              data-cy="team-name"
              :disabled="disabled"
              :error-messages="nameErrors"
              :loading="isCheckingName"
              class="white--text v-text-field-input-color"
              color="white"
              @blur="checkName(name)"
              @input="resetNameMetadata"
            >
              <v-icon
                v-if="showNameIcon && nameErrors.length === 0"
                slot="append"
                class="white--text"
              >
                check
              </v-icon>
              <v-icon
                v-if="showNameIcon && nameErrors.length > 0"
                slot="append"
                class="red--text"
              >
                clear
              </v-icon>
            </v-text-field>
          </v-col>
          <v-col
            v-if="revealUrlInput"
            key="input-2"
            cols="12"
            class="my-2 mb-12 name-team-input mx-auto"
          >
            <div class="overline">
              Team URL
            </div>
            <div v-if="tenant.role !== 'TENANT_ADMIN'" class="headline medium">
              {{ tenant.slug }}
            </div>
            <v-text-field
              v-if="tenant.role == 'TENANT_ADMIN'"
              v-model="slug"
              data-cy="team-slug"
              :disabled="disabled"
              :error-messages="slugErrors"
              :loading="isCheckingSlug"
              class="white--text v-text-field-input-color"
              color="white"
              @blur="checkSlug(slug)"
              @input="resetSlugMetadata"
            >
              <v-icon
                v-if="showSlugIcon && slugErrors.length === 0"
                slot="append"
                class="white--text"
              >
                check
              </v-icon>
              <v-icon
                v-if="showSlugIcon && slugErrors.length > 0"
                slot="append"
                class="red--text"
              >
                clear
              </v-icon>
            </v-text-field>
          </v-col>

          <v-col
            v-if="revealConfirm"
            key="revealConfirm"
            cols="12"
            class="my-2"
          >
            <v-btn
              v-if="tenant.role == 'TENANT_ADMIN'"
              color="primary"
              width="auto"
              data-cy="submit-team-info"
              :loading="loading > 0"
              :disabled="disabled"
              @click="updateTenant"
            >
              Next
              <v-icon right>arrow_right</v-icon>
            </v-btn>
          </v-col>

          <v-col
            v-if="
              updateServerError &&
                nameErrors.length === 0 &&
                slugErrors.length === 0
            "
            key="error"
            cols="12"
            class="body-2 red--text text--darken-1"
          >
            Sorry, something went wrong. Please try again.
          </v-col>
        </transition-group>
      </div>
    </v-row>
  </v-card>
</template>

<style lang="scss" scoped>
.h-80 {
  min-height: calc(80vh - 64px) !important;
}

.transition-height {
  transition: max-height 500ms ease;
}

.name-team-input {
  max-width: 700px;
}

.w-100 {
  width: 100vw;
}

.name-team-card {
  left: 50%;
  top: 50%;
  transform: translate(-50%, -50%);
}
</style>
