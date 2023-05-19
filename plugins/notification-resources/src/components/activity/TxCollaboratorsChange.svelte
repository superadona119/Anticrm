<!--
// Copyright © 2023 Hardcore Engineering Inc.
//
// Licensed under the Eclipse Public License, Version 2.0 (the "License");
// you may not use this file except in compliance with the License. You may
// obtain a copy of the License at https://www.eclipse.org/legal/epl-2.0
//
// Unless required by applicable law or agreed to in writing, software
// distributed under the License is distributed on an "AS IS" BASIS,
// WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
//
// See the License for the specific language governing permissions and
// limitations under the License.
-->
<script lang="ts">
  import { EmployeeAccount } from '@hcengineering/contact'
  import { EmployeeAccountRefPresenter } from '@hcengineering/contact-resources'
  import { Doc, Ref, TxMixin } from '@hcengineering/core'
  import { Collaborators } from '@hcengineering/notification'
  import { getClient } from '@hcengineering/presentation'
  import { IconAdd, IconDelete, Label } from '@hcengineering/ui'
  import notification from '../../plugin'

  export let tx: TxMixin<Doc, Collaborators>
  export let value: Collaborators
  export let prevValue: Collaborators | undefined = undefined

  interface Diff {
    added: Ref<EmployeeAccount>[]
    removed: Ref<EmployeeAccount>[]
  }
  const client = getClient()
  const hierarchy = client.getHierarchy()

  function buildDiff (value: Collaborators, prev: Collaborators | undefined): Diff | undefined {
    if (prev === undefined) return
    const added: Ref<EmployeeAccount>[] = []
    const removed: Ref<EmployeeAccount>[] = []
    const mixin = hierarchy.as(value, notification.mixin.Collaborators)
    const prevMixin = hierarchy.as(prev, notification.mixin.Collaborators)
    const prevSet = new Set(prevMixin?.collaborators ?? [])
    const newSet = new Set(mixin.collaborators)

    for (const newCollab of mixin.collaborators) {
      if (!prevSet.has(newCollab)) added.push(newCollab as Ref<EmployeeAccount>)
    }

    for (const oldCollab of prevMixin?.collaborators ?? []) {
      if (!newSet.has(oldCollab)) removed.push(oldCollab as Ref<EmployeeAccount>)
    }

    return {
      added,
      removed
    }
  }

  $: diff = buildDiff(value, prevValue)
</script>

{#if diff}
  <div class="flex-presenter">
    <Label label={notification.string.ChangeCollaborators} />
    {#if diff.added.length > 0}
      <IconAdd size={'x-small'} fill={'var(--theme-trans-color)'} />
      {#each diff.added as add}
        <EmployeeAccountRefPresenter value={add} disabled />
      {/each}
    {/if}
    {#if diff.removed.length > 0}
      <IconDelete size={'x-small'} fill={'var(--theme-trans-color)'} />
      {#each diff.removed as removed}
        <EmployeeAccountRefPresenter value={removed} disabled />
      {/each}
    {/if}
  </div>
{:else}
  <Label label={notification.string.YouAddedCollaborators} />
{/if}