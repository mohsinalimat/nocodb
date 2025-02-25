<template>
  <div>
    <v-toolbar flat height="42" class="toolbar-border-bottom">
      <v-toolbar-title>
        <v-breadcrumbs
          :items="[{
                     text: nodes.env,
                     disabled: true,
                     href: '#'
                   },{
                     text: nodes.dbAlias,
                     disabled: true,
                     href: '#'
                   },
                   {
                     text: nodes._tn + ' (Webhooks)',
                     disabled: true,
                     href: '#'
                   }]"
          divider=">"
          small
        >
          <template #divider>
            <v-icon small color="grey lighten-2">
              forward
            </v-icon>
          </template>
        </v-breadcrumbs>
      </v-toolbar-title>
      <v-spacer />

      <x-btn
        v-ge="['hooks','reload']"
        outlined
        tooltip="Reload hooks"
        color="primary"
        small
        @click.prevent="loadHooksList"
      >
        <v-icon small left>
          mdi-reload
        </v-icon>
        Reload
      </x-btn>

      <x-btn
        v-ge="['hooks','add new']"
        outlined
        tooltip="Save Changes"
        color="primary"
        small
        @click.prevent="addNewHook"
      >
        <v-icon small left>
          mdi-plus
        </v-icon>
        Add New
      </x-btn>

      <!--      <x-btn outlined tooltip="Save Changes"
                   color="primary"
                   small
                   :disabled="loading || !valid || !hook.event"

                   v-ge="['hooks','save']"
                   @click.prevent="saveHooks">
              <v-icon small left>save</v-icon>
              Save
            </x-btn>-->
    </v-toolbar>

    <v-form
      ref="form"
      v-model="valid"
      class="mx-auto"
      lazy-validation
    >
      <v-card>
        <v-container fluid>
          <v-row>
            <v-col cols="7">
              <v-radio-group v-model="selectedHook" @change="onEventChange">
                <template #default>
                  <v-simple-table dense>
                    <template #default>
                      <thead>
                        <tr>
                          <th />
                          <th>Title</th>
                          <th>Event</th>
                          <th>Condition</th>
                          <th>Notify Via</th>
                          <th>Action</th>
                        </tr>
                      </thead>

                      <tbody>
                        <template v-if="hooks && hooks.length">
                          <tr v-for="(item,i) in hooks" :key="i">
                            <td>
                              <v-radio :value="i" />
                            </td>
                            <td>{{ item.title }}</td>
                            <td>{{ item.event }} {{ item.operation }}</td>
                            <td>
                              <v-icon v-if="item.condition" color="success" small>
                                mdi-check-bold
                              </v-icon>
                            </td>
                            <td>{{ item.notification && item.notification.type }}</td>
                            <td>
                              <x-icon small color="error" @click.stop="deleteHook(item, i)">
                                mdi-delete
                              </x-icon>
                            <!--              <x-icon small :color="loading || !valid || !hook.event ?  'grey' : 'primary'"
                                                  @click.stop="(!loading && valid && hook.event) && saveHooks()">save
                                          </x-icon>-->
                            </td>
                          </tr>
                        </template>
                        <tr v-else>
                          <td colspan="6" class="text-center py-5">
                            <x-btn
                              v-ge="['hooks','add new']"
                              outlined
                              tooltip="Save Changes"
                              color="primary"
                              small
                              @click.prevent="addNewHook"
                            >
                              <v-icon small left>
                                mdi-plus
                              </v-icon>
                              Add New Webhook
                            </x-btn>
                          </td>
                        </tr>
                      </tbody>
                    </template>
                  </v-simple-table>
                </template>
              </v-radio-group>
            </v-col>
            <v-col cols="5">
              <v-card v-if="hook" class="">
                <v-card-title>
                  Webhook
                  <v-spacer />
                  <x-btn
                    v-ge="['hooks','save']"
                    outlined
                    tooltip="Save"
                    color="primary"
                    small

                    :disabled="loading || !valid || !hook.event"
                    @click.prevent="saveHooks"
                  >
                    <v-icon small left>
                      save
                    </v-icon>
                    Save
                  </x-btn>
                </v-card-title>
                <v-card-text>
                  <v-text-field
                    v-model="hook.title"
                    class="caption"
                    outlined
                    dense
                    label="Title"
                    required
                    :rules="[v => !!v || 'Title Required']"
                  />

                  <webhook-event
                    :event.sync="hook.event"
                    :operation.sync="hook.operation"
                  />

                  <v-card class="mb-8">
                    <v-card-text>
                      <v-checkbox
                        v-model="enableCondition"
                        dense
                        hide-details
                        class="mt-1"
                        label="On Condition"
                        @change="checkConditionAvail"
                      />

                      <column-filter
                        v-if="enableCondition && _isEE"
                        v-model="hook.condition"
                        :field-list="fieldList"
                        dense
                        style="max-width: 100%"
                      />
                    </v-card-text>
                  </v-card>

                  <v-select
                    v-model="hook.notification.type"
                    outlined
                    dense
                    label="Notification"
                    required
                    :items="notificationList"
                    :rules="[v => !!v || 'Title Required']"
                    class="caption"
                    :prepend-inner-icon="notificationIcon[hook.notification.type]"
                    @change="onNotTypeChange"
                  >
                    <template #item="{item}">
                      <v-list-item-icon>
                        <v-icon small>
                          {{ notificationIcon[item] }}
                        </v-icon>
                      </v-list-item-icon>
                      <v-list-item-title>
                        {{ item }}
                      </v-list-item-title>
                    </template>
                  </v-select>

                  <template v-if="hook.notification.type === 'URL'">
                    <http-webhook v-model="notification" />
                  </template>

                  <template v-if="hook.notification.type === 'Slack'">
                    <v-combobox
                      v-if="slackChannels"
                      v-model="notification.channels"
                      :rules="[v => !!v || 'Required']"
                      :items="slackChannels"
                      item-text="channel"
                      label="Select Slack channels"
                      multiple
                      outlined
                      dense
                      class="caption"
                    />
                  </template>
                  <template v-if="hook.notification.type === 'Microsoft Teams'">
                    <v-combobox
                      v-if="teamsChannels"
                      v-model="notification.channels"
                      :rules="[v => !!v || 'Required']"
                      :items="teamsChannels"
                      item-text="channel"
                      label="Select Teams channels"
                      multiple
                      outlined
                      dense
                      class="caption"
                    />
                  </template>
                  <template v-if="hook.notification.type === 'Discord'">
                    <v-combobox
                      v-if="discordChannels"
                      v-model="notification.channels"
                      :rules="[v => !!v || 'Required']"
                      :items="discordChannels"
                      item-text="channel"
                      label="Select Discord channels"
                      multiple
                      outlined
                      dense
                      class="caption"
                    />
                  </template>
                  <template v-if="hook.notification.type === 'Mattermost'">
                    <v-combobox
                      v-if="mattermostChannels"
                      v-model="notification.channels"
                      :rules="[v => !!v || 'Required']"
                      :items="mattermostChannels"
                      item-text="channel"
                      label="Select Mattermost channels"
                      multiple
                      outlined
                      dense
                      class="caption"
                    />
                  </template>

                  <template v-if="inputs[hook.notification.type] && notification">
                    <template v-for="input in inputs[hook.notification.type]">
                      <v-textarea
                        v-if="input.type === 'LongText'"
                        :key="input.key"
                        v-model="notification[input.key]"
                        class="caption"
                        dense
                        outlined
                        :label="input.label"
                        :rules="[v => !input.required || !!v || 'Required']"
                      />
                      <v-text-field
                        v-else
                        :key="input.key"
                        v-model="notification[input.key]"
                        class="caption"
                        dense
                        outlined
                        :label="input.label"
                        :rules="[v => !input.required || !!v || 'Required']"
                      />
                    </template>
                  </template>
                </v-card-text>

                <v-card-text>
                  <span class="caption grey--text">
                    <em>Available context variables are <strong>data, user, payload and env</strong></em>
                    <v-tooltip top>
                      <template #activator="{on}">
                        <v-icon
                          small
                          color="grey"
                          class="ml-2"
                          v-on="on"
                        >mdi-information</v-icon>
                      </template>
                      <span class="caption">
                        <strong>data</strong> : Row data <br>
                        <strong>user</strong> : User information<br>
                        <strong>payload</strong> : Plugin settings payload<br>
                        <strong>env</strong> : Environment values (process.env)

                      </span>
                    </v-tooltip>
                  </span>
                </v-card-text>
              </v-card>
            </v-col>
          </v-row>
        </v-container>
      </v-card>
    </v-form>
  </div>
</template>

<script>
import HttpWebhook from './webhook/httpWebhook'
import ColumnFilter from '~/components/project/spreadsheet/components/columnFilter'
// import FormInput from '~/components/project/appStore/FormInput'
import WebhookEvent from '~/components/project/tableTabs/webhookEvent'

export default {
  name: 'Webhooks',
  components: {
    HttpWebhook,
    WebhookEvent,
    // FormInput,
    ColumnFilter
  },
  props: ['nodes'],
  data: () => ({
    slackChannels: null,
    teamsChannels: null,
    discordChannels: null,
    mattermostChannels: null,
    selectedHook: null,
    enableCondition: false,
    hooks: null,
    valid: false,
    loading: false,
    notificationList: [
      'Email',
      'Slack',
      'Microsoft Teams',
      'Discord',
      'Mattermost',
      'Twilio',
      'Whatsapp Twilio',
      'URL'
    ],
    filters: [],
    hook: null,
    notification: {},
    notificationIcon: {
      URL: 'mdi-link',
      Email: 'mdi-email',
      Slack: 'mdi-slack',
      'Microsoft Teams': 'mdi-microsoft-teams',
      Discord: 'mdi-discord',
      Mattermost: 'mdi-chat',
      'Whatsapp Twilio': 'mdi-whatsapp',
      Twilio: 'mdi-cellphone-message'
    },
    urlRules: [
      v => !v || !v.trim() || /^https?:\/\/.{1,}/.test(v) || 'Not a valid URL'
    ],
    fieldList: [],
    inputs: {
      Email: [
        {
          key: 'to',
          label: 'To Address',
          placeholder: 'To Address',
          type: 'SingleLineText',
          required: true
        },
        {
          key: 'subject',
          label: 'Subject',
          placeholder: 'Subject',
          type: 'SingleLineText',
          required: true
        }, {
          key: 'body',
          label: 'Body',
          placeholder: 'Body',
          type: 'LongText',
          required: true
        }
      ],
      Slack: [{
        key: 'body',
        label: 'Body',
        placeholder: 'Body',
        type: 'LongText',
        required: true
      }
      ],
      'Microsoft Teams': [{
        key: 'body',
        label: 'Body',
        placeholder: 'Body',
        type: 'LongText',
        required: true
      }
      ],
      Discord: [{
        key: 'body',
        label: 'Body',
        placeholder: 'Body',
        type: 'LongText',
        required: true
      }
      ],
      Mattermost: [{
        key: 'body',
        label: 'Body',
        placeholder: 'Body',
        type: 'LongText',
        required: true
      }
      ],
      Twilio: [{
        key: 'body',
        label: 'Body',
        placeholder: 'Body',
        type: 'LongText',
        required: true
      }, {
        key: 'to',
        label: 'Comma separated Mobile #',
        placeholder: 'Comma separated Mobile #',
        type: 'LongText',
        required: true
      }],
      'Whatsapp Twilio': [{
        key: 'body',
        label: 'Body',
        placeholder: 'Body',
        type: 'LongText',
        required: true
      }, {
        key: 'to',
        label: 'Comma separated Mobile #',
        placeholder: 'Comma separated Mobile #',
        type: 'LongText',
        required: true
      }]
    }
  }),
  async created() {
    await this.loadMeta()
    await this.loadHooksList()
    this.selectedHook = 0
    this.onEventChange()
  },
  methods: {
    checkConditionAvail() {
      if (!process.env.EE) {
        this.enableCondition = false
        this.$toast.info('For webhook condition : Upgrade to Enterprise Edition').goAway(3000)
      }
      this.hook.condition = []
    },
    async onNotTypeChange() {
      this.notification = {}
      if (this.hook.notification.type === 'Slack') {
        const plugin = await this.$store.dispatch('sqlMgr/ActSqlOp', [null, 'xcPluginRead', {
          title: 'Slack'
        }])
        this.slackChannels = JSON.parse(plugin.input) || []
      }
      if (this.hook.notification.type === 'Microsoft Teams') {
        const plugin = await this.$store.dispatch('sqlMgr/ActSqlOp', [null, 'xcPluginRead', {
          title: 'Microsoft Teams'
        }])
        this.teamsChannels = JSON.parse(plugin.input) || []
      }
      if (this.hook.notification.type === 'Discord') {
        const plugin = await this.$store.dispatch('sqlMgr/ActSqlOp', [null, 'xcPluginRead', {
          title: 'Discord'
        }])
        this.discordChannels = JSON.parse(plugin.input) || []
      }
      if (this.hook.notification.type === 'Mattermost') {
        const plugin = await this.$store.dispatch('sqlMgr/ActSqlOp', [null, 'xcPluginRead', {
          title: 'Mattermost'
        }])
        this.mattermostChannels = JSON.parse(plugin.input) || []
      }
    },
    async onEventChange() {
      if (!this.hooks || !this.hooks.length) {
        return
      }
      const { notification: { payload, type }, ...hook } = this.hooks[this.selectedHook]

      this.hook = {
        ...hook,
        notification: {
          type
        }
      }
      this.enableCondition = !!(this.hook && this.hook.condition && Object.keys(this.hook.condition).length)
      await this.onNotTypeChange()
      this.notification = payload
      if (this.hook.notification.type === 'Slack') {
        this.notification.webhook_url = this.notification.webhook_url &&
          this.notification.webhook_url.map(v => this.slackChannels.find(s => v.webhook_url === s.webhook_url))
      }
      if (this.hook.notification.type === 'Microsoft Teams') {
        this.notification.webhook_url = this.notification.webhook_url &&
          this.notification.webhook_url.map(v => this.teamsChannels.find(s => v.webhook_url === s.webhook_url))
      }
      if (this.hook.notification.type === 'Discord') {
        this.notification.webhook_url = this.notification.webhook_url &&
          this.notification.webhook_url.map(v => this.discordChannels.find(s => v.webhook_url === s.webhook_url))
      }
      if (this.hook.notification.type === 'Mattermost') {
        this.notification.webhook_url = this.notification.webhook_url &&
          this.notification.webhook_url.map(v => this.mattermostChannels.find(s => v.webhook_url === s.webhook_url))
      }
      if (this.hook.notification.type === 'URL') {
        // eslint-disable-next-line no-self-assign
        this.notification.api = this.notification.api
      }
    },
    async saveHooks() {
      if (!this.$refs.form.validate() || !this.valid || !this.hook.event) {
        return
      }
      this.loading = true
      try {
        const res = await this.$store.dispatch('sqlMgr/ActSqlOp', [
          {
            env: this.nodes.env,
            dbAlias: this.nodes.dbAlias
          }, 'tableXcHooksSet', {
            tn: this.nodes.tn,
            data: {
              ...this.hook,
              notification: {
                ...this.hook.notification,
                payload: this.notification
              }
            }
          }
        ])

        if (!this.hook.id && res) {
          this.hook.id = Array.isArray(res) ? res[0] : res
        }

        this.$toast.success('Webhook details updated successfully').goAway(3000)
      } catch (e) {
        this.$toast.error(e.message).goAway(3000)
      }
      this.loading = false
      await this.loadHooksList()
    },
    async loadMeta() {
      this.loadingMeta = true
      const tableMeta = await this.$store.dispatch('sqlMgr/ActSqlOp', [{
        env: this.nodes.env,
        dbAlias: this.nodes.dbAlias
      }, 'tableXcModelGet', {
        tn: this.nodes.tn
      }])
      this.meta = JSON.parse(tableMeta.meta)
      this.fieldList = this.meta.columns.map(c => c.cn)
      this.loadingMeta = false
    },
    async loadHooksList() {
      this.loading = true
      const hooks = await this.$store.dispatch('sqlMgr/ActSqlOp', [{
        env: this.nodes.env,
        dbAlias: this.nodes.dbAlias
      }, 'tableXcHooksList', {
        tn: this.nodes.tn
      }])
      this.hooks = hooks.data.list.map((h) => {
        h.notification = h.notification && JSON.parse(h.notification)
        h.condition = h.condition && JSON.parse(h.condition)

        return h
      })
      this.loading = false
    },
    addNewHook() {
      this.selectedHook = this.hooks.length
      this.hooks.push({
        notification: {
          // type:'Email'
        }
      })
      this.onEventChange()
      this.$refs.form.resetValidation()
    },
    async deleteHook(item, i) {
      try {
        if (item.id) {
          await this.$store.dispatch('sqlMgr/ActSqlOp', [{
            env: this.nodes.env,
            dbAlias: this.nodes.dbAlias
          }, 'tableXcHooksDelete', {
            id: item.id,
            title: item.title,
            tn: this.nodes.tn
          }])
          this.hooks.splice(i, 1)
        } else {
          this.hooks.splice(i, 1)
        }
        this.$toast.success('Hook deleted successfully').goAway(3000)
        if (!this.hooks.length) {
          this.hook = null
        }
      } catch (e) {
        this.$toast.error(e.message).goAway(3000)
      }
    }
  }
}
</script>

<style scoped>
/*tr.selected{*/
/*  background: #8ceaf6;*/
/*}*/

/deep/ label {
  font-size: 0.75rem !important
}
</style>
<!--
/**
 * @copyright Copyright (c) 2021, Xgene Cloud Ltd
 *
 * @author Naveen MR <oof1lab@gmail.com>
 * @author Pranav C Balan <pranavxc@gmail.com>
 *
 * @license GNU AGPL version 3 or any later version
 *
 * This program is free software: you can redistribute it and/or modify
 * it under the terms of the GNU Affero General Public License as
 * published by the Free Software Foundation, either version 3 of the
 * License, or (at your option) any later version.
 *
 * This program is distributed in the hope that it will be useful,
 * but WITHOUT ANY WARRANTY; without even the implied warranty of
 * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
 * GNU Affero General Public License for more details.
 *
 * You should have received a copy of the GNU Affero General Public License
 * along with this program. If not, see <http://www.gnu.org/licenses/>.
 *
 */
-->
