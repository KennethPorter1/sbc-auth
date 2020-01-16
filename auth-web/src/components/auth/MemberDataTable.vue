<template>
  <v-container class="view-container">
  <v-data-table
    class="user-list"
    :headers="headerMembers"
    :items="indexedOrgMembers"
    :items-per-page="5"
    :hide-default-footer="indexedOrgMembers.length <= 5"
    :custom-sort="customSortActive"
    :no-data-text="$t('noActiveUsersLabel')"
  >
    <template v-slot:loading>
      Loading...
    </template>
    <template v-slot:item.name="{ item }">
      <v-list-item-title class="user-name" :data-test="getIndexedTag('user-name', item.index)">{{ item.user.firstname }} {{ item.user.lastname }}</v-list-item-title>
      <v-list-item-subtitle :data-test="getIndexedTag('business-id', item.index)" v-if="item.user.contacts && item.user.contacts.length > 0">{{ item.user.contacts[0].email }}</v-list-item-subtitle>
    </template>
    <template v-slot:item.role="{ item }">
      <v-menu>
        <template v-slot:activator="{ on }">
          <v-btn :disabled="!canChangeRole(item)" class="role-selector" small depressed v-on="on" :data-test="getIndexedTag('role-selector', item.index)">
            {{ item.membershipTypeCode }}
            <v-icon small depressed class="ml-1">mdi-chevron-down</v-icon>
          </v-btn>
        </template>
        <v-list dense class="role-list">
          <v-item-group>
            <v-list-item three-line
              v-for="(role, index) in availableRoles"
              :key="index"
              @click="item.membershipTypeCode.toUpperCase() !== role.name.toUpperCase()? confirmChangeRole(item, role.name): ''"
              :disabled="!isRoleEnabled(role)"
              v-bind:class="{'primary--text v-item--active v-list-item--active': item.membershipTypeCode.toUpperCase() === role.name.toUpperCase()}">
              <v-list-item-icon>
                <v-icon v-text="role.icon" />
              </v-list-item-icon>
              <v-list-item-content>
                <v-list-item-title
                  v-text="role.name"
                >
                </v-list-item-title>
                <v-list-item-subtitle
                  v-text="role.desc"
                >
                </v-list-item-subtitle>
                <v-divider></v-divider>
              </v-list-item-content>
            </v-list-item>
          </v-item-group>
        </v-list>
      </v-menu>

    </template>
    <template v-slot:item.lastActive="{ item }" :data-test="getIndexedTag('last-active', item.index)">
      {{ formatDate(item.user.modified) }}
    </template>
    <template v-slot:item.action="{ item }">
      <v-btn :data-test="getIndexedTag('remove-user-button', item.index)" v-show="canRemove(item)" depressed small @click="confirmRemoveMember(item)">Remove</v-btn>
      <v-btn :data-test="getIndexedTag('leave-team-button', item.index)" v-show="canLeave(item)" depressed small @click="confirmLeaveTeam(item)">
        <span v-if="!canDissolve()">Leave</span>
        <span v-if="canDissolve()">Dissolve</span>
      </v-btn>
    </template>
  </v-data-table>

    <ModalDialog
            ref="confirmActionDialog"
            :title="confirmActionTitle"
            :text="confirmActionText"
            dialog-class="notify-dialog"
            max-width="640"
    >
      <template v-slot:icon>
        <v-icon large color="error">mdi-alert-circle-outline</v-icon>
      </template>
      <template v-slot:actions>
        <v-btn large color="primary" @click="confirmHandler()">{{ primaryActionText }}</v-btn>
        <v-btn large color="default" @click="cancel()">{{ secondaryActionText }}</v-btn>
      </template>
    </ModalDialog>

    <!-- Confirm Action Dialog With Email Question-->
    <ModalDialog
            ref="confirmActionDialogWithQuestion"
            :title="confirmActionTitle"
            :text="confirmActionText"
            dialog-class="notify-dialog"
            max-width="640"
    >
      <template v-slot:icon>
        <v-icon large color="primary">mdi-information-outline</v-icon>
      </template>
      <template v-slot:text>
        {{ confirmActionText }}
      </template>
      <template v-slot:actions>
        <v-btn large color="primary" @click="confirmHandler()">{{ primaryActionText }}</v-btn>
        <v-btn large color="default" @click="cancelEmailModal()">{{ secondaryActionText }}</v-btn>
      </template>
    </ModalDialog>

    <!-- Alert Dialog (Success) -->
    <ModalDialog
            ref="successDialog"
            :title="successTitle"
            :text="successText"
            dialog-class="notify-dialog"
            max-width="640"
    ></ModalDialog>

    <!-- Alert Dialog (Error) -->
    <ModalDialog
            ref="errorDialog"
            :title="errorTitle"
            :text="errorText"
            dialog-class="notify-dialog"
            max-width="640"
    >
      <template v-slot:icon>
        <v-icon large color="error">mdi-alert-circle-outline</v-icon>
      </template>
      <template v-slot:actions>
        <v-btn large color="error" @click="close()">OK</v-btn>
      </template>
    </ModalDialog>

  </v-container>
</template>

<script lang="ts">
import { Component, Emit, Vue } from 'vue-property-decorator'
import { Member, MembershipStatus, MembershipType, Organization, RoleInfo, UpdateMemberPayload } from '@/models/Organization'
import { mapActions, mapGetters, mapState } from 'vuex'
import { Business } from '@/models/business'
import ModalDialog from '@/components/auth/ModalDialog.vue'
import moment from 'moment'

export interface ChangeRolePayload {
  member: Member;
  targetRole: string;
}

@Component({
  components: {
    ModalDialog
  },
  computed: {
    ...mapState('business', ['businesses']),
    ...mapState('org', ['activeOrgMembers', 'myOrgMembership'])
  },
  methods: {
    ...mapActions('org', [
      'resendInvitation',
      'deleteInvitation',
      'updateMember',
      'approveMember',
      'leaveTeam',
      'syncOrganizations',
      'syncActiveOrgMembers'
    ])
  }
})
export default class MemberDataTable extends Vue {
  private readonly businesses!: Business[]
  private readonly activeOrgMembers!: Member[]
  private readonly myOrgMembership!: Member
  private readonly syncActiveOrgMembers!: () => Member[]
  private confirmActionTitle: string = ''
  private confirmActionText: string = ''
  private primaryActionText: string = ''
  private secondaryActionText = 'No'
  private confirmHandler: () => void = undefined
  private memberToBeRemoved: Member
  private readonly updateMember!: (updateMemberPayload: UpdateMemberPayload) => void
  private roleChangeToAction: ChangeRolePayload
  private notifyUser = true
  private readonly leaveTeam!: (memberId: number) => void
  private readonly syncOrganizations!: () => Promise<Organization[]>
  private errorTitle: string = ''
  private errorText: string = ''
  private successTitle: string = ''
  private successText: string = ''

  $refs: {
    errorDialog: ModalDialog
    confirmActionDialog: ModalDialog
    confirmActionDialogWithQuestion:ModalDialog
  }

  private readonly availableRoles: RoleInfo[] = [
    {
      icon: 'mdi-account',
      name: 'Member',
      desc: 'Can add businesses, and file for a business.'
    },
    {
      icon: 'mdi-settings',
      name: 'Admin',
      desc: 'Can add/remove team members, add businesses, and file for a business.'
    },
    {
      icon: 'mdi-shield-key',
      name: 'Owner',
      desc: 'Can add/remove team members and businesses, and file for a business.'
    }
  ]

  private readonly headerMembers = [
    {
      text: 'Team Member',
      align: 'left',
      sortable: true,
      value: 'name'
    },
    {
      text: 'Roles',
      align: 'left',
      sortable: true,
      value: 'role'
    },
    {
      text: 'Last Active',
      align: 'left',
      sortable: true,
      value: 'lastActive'
    },
    {
      text: 'Actions',
      align: 'left',
      value: 'action',
      sortable: false,
      width: '80'
    }
  ]

  private async mounted () {
    await this.syncActiveOrgMembers()
  }

  private getIndexedTag (tag, index): string {
    return `${tag}-${index}`
  }

  private get indexedOrgMembers () {
    return this.activeOrgMembers.map((item, index) => ({
      index,
      ...item
    }))
  }

  private formatDate (date: Date) {
    return moment(date).format('DD MMM, YYYY')
  }

  private isRoleEnabled (role: RoleInfo): boolean {
    switch (this.myOrgMembership.membershipTypeCode) {
      case MembershipType.Owner:
        return true
      case MembershipType.Admin:
        if (role.name !== 'Owner') {
          return true
        }
        return false
      default:
        return false
    }
  }

  private canChangeRole (memberBeingChanged: Member): boolean {
    if (this.myOrgMembership.membershipStatus !== MembershipStatus.Active) {
      return false
    }

    switch (this.myOrgMembership.membershipTypeCode) {
      case MembershipType.Owner:
        // Owners can change roles of other users who are not owners
        if (!this.isOwnMembership(memberBeingChanged) && memberBeingChanged.membershipTypeCode !== MembershipType.Owner) {
          return true
        }
        // And they can downgrade their own role if there is another owner on the team
        if (this.isOwnMembership(memberBeingChanged) && this.ownerCount() > 1) {
          return true
        }
        return false
      case MembershipType.Admin:
        // Admins can change roles of other admins and members, including their own
        if (memberBeingChanged.membershipTypeCode !== MembershipType.Owner) {
          return true
        }
        return false
      default:
        return false
    }
  }

  private canRemove (memberToRemove: Member): boolean {
    // Can't remove yourself
    if (this.myOrgMembership.user.username === memberToRemove.user.username) {
      return false
    }

    // Can't remove unless Admin/Owner
    if (this.myOrgMembership.membershipTypeCode === MembershipType.Member) {
      return false
    }

    // Can't remove Owners unless an Owner
    if (memberToRemove.membershipTypeCode === MembershipType.Owner &&
        this.myOrgMembership.membershipTypeCode !== MembershipType.Owner) {
      return false
    }

    return true
  }

  private canLeave (member: Member): boolean {
    if (this.myOrgMembership.user.username !== member.user.username) {
      return false
    }
    return true
  }

  private ownerCount (): number {
    return this.activeOrgMembers.filter(member => member.membershipTypeCode === MembershipType.Owner).length
  }

  private customSortActive (items, index, isDescending) {
    const isDesc = isDescending.length > 0 && isDescending[0]
    switch (index[0]) {
      case 'name':
        items.sort((a, b) => {
          if (isDesc) {
            return a.user.firstname < b.user.firstname ? -1 : 1
          } else {
            return b.user.firstname < a.user.firstname ? -1 : 1
          }
        })
        break
      case 'role':
        items.sort((a, b) => {
          if (isDesc) {
            return a.membershipTypeCode < b.membershipTypeCode ? -1 : 1
          } else {
            return b.membershipTypeCode < a.membershipTypeCode ? -1 : 1
          }
        })
        break
      case 'lastActive':
        items.sort((a, b) => {
          if (isDesc) {
            return a.user.modified < b.user.modified ? -1 : 1
          } else {
            return b.user.modified < a.user.modified ? -1 : 1
          }
        })
    }
    return items
  }

  private canDissolve (): boolean {
    if (this.activeOrgMembers.length === 1 && this.businesses.length === 0) {
      return true
    }
    return false
  }

  private close () {
    this.$refs.errorDialog.close()
  }

  @Emit()
  private confirmRemoveMember (member: Member) {
    if (member.membershipStatus === MembershipStatus.Pending) {
      this.confirmActionTitle = this.$t('confirmDenyMemberTitle').toString()
      this.confirmActionText = `Are you sure you want to deny membership to ${member.user.firstname}?`
      this.confirmHandler = this.deny
      this.primaryActionText = 'Deny'
    } else {
      this.confirmActionTitle = this.$t('confirmRemoveMemberTitle').toString()
      this.confirmActionText = `Are you sure you want to remove ${member.user.firstname} from the team?`
      this.confirmHandler = this.removeMember
      this.primaryActionText = 'Remove'
    }
    this.memberToBeRemoved = member
    this.$refs.confirmActionDialog.open()
  }

  @Emit()
  private confirmChangeRole (member: Member, targetRole: string): ChangeRolePayload {
    if (member.membershipTypeCode.toString() === targetRole.toString()) {
      return
    }
    this.confirmActionTitle = this.$t('confirmRoleChangeTitle').toString()
    this.confirmActionText = `Are you sure you wish to change ${member.user.firstname}'s role to ${targetRole}?`
    this.roleChangeToAction = {
      member,
      targetRole
    }
    this.confirmHandler = this.changeRole
    this.primaryActionText = 'Yes'
    this.$refs.confirmActionDialogWithQuestion.open()

    return {
      member,
      targetRole
    }
  }

  private async changeRole () {
    await this.updateMember({
      memberId: this.roleChangeToAction.member.id,
      role: this.roleChangeToAction.targetRole.toString().toUpperCase(),
      notifyUser: this.notifyUser
    })
    this.$refs.confirmActionDialogWithQuestion.close()
    await this.syncOrganizations()
  }

  private confirmLeaveTeam (member: Member) {
    if (member.membershipTypeCode === MembershipType.Owner &&
        this.ownerCount() === 1 &&
        !this.canDissolve()) {
      this.confirmActionTitle = this.$t('confirmLeaveTeamTitle').toString()
      this.confirmActionText = this.$t('confirmLeaveTeamText').toString()
      this.confirmHandler = this.leave
      this.primaryActionText = 'Leave'
      this.$refs.confirmActionDialog.open()
    } else {
      this.confirmActionTitle = this.$t('confirmLeaveTeamTitle').toString()
      this.confirmActionText = this.$t('confirmLeaveTeamText').toString()
      this.confirmHandler = this.leave
      this.primaryActionText = 'Leave'
      this.$refs.confirmActionDialog.open()
    }
  }

  private async leave () {
    await this.leaveTeam(this.myOrgMembership.id)
    this.$refs.confirmActionDialog.close()
    this.$router.push('/leaveteam')
  }

  private isOwnMembership (member: Member) {
    return !!this.myOrgMembership && this.myOrgMembership.user.username === member.user.username
  }
  private async deny () {
    await this.updateMember({
      memberId: this.memberToBeRemoved.id,
      status: MembershipStatus.Rejected
    })
    this.$refs.confirmActionDialog.close()
  }

  private cancel () {
    this.$refs.confirmActionDialog.close()
  }

  private async removeMember () {
    await this.updateMember({
      memberId: this.memberToBeRemoved.id,
      status: MembershipStatus.Inactive
    })
    this.$refs.confirmActionDialog.close()
  }
}
</script>

<style lang="scss" scoped>
  @import "$assets/scss/theme.scss";

  .v-list--dense {
    .v-list-item {
      padding-top: 0.25rem;
      padding-bottom: 0.25rem;
    }

    .v-list-item .v-list-item__title {
      margin-bottom: 0.25rem;
      font-weight: 700;
    }
  }

  .role-list {
    width: 20rem;
  }
</style>
