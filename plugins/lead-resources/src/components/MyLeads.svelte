<!--
// Copyright © 2022 Hardcore Engineering Inc.
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
  import { AttachedDoc, Class, DocumentQuery, getCurrentAccount, Ref } from '@hcengineering/core'
  import { createQuery, getClient } from '@hcengineering/presentation'
  import task, { Task } from '@hcengineering/task'
  import { IModeSelector, Label, resolvedLocationStore, SearchEdit, ModeSelector, Loading } from '@hcengineering/ui'
  import {
    FilterButton,
    getViewOptions,
    makeViewletKey,
    TableBrowser,
    viewOptionStore
  } from '@hcengineering/view-resources'
  import { IntlString } from '@hcengineering/platform'
  import ViewletSettingButton from '@hcengineering/view-resources/src/components/ViewletSettingButton.svelte'
  import view, { Viewlet, ViewletPreference } from '@hcengineering/view'
  import { onDestroy } from 'svelte'
  import FilterBar from '@hcengineering/view-resources/src/components/filter/FilterBar.svelte'
  import lead from '../plugin'
  import { Lead } from '@hcengineering/lead'

  export let _class: Ref<Class<Lead>> = lead.class.Lead
  export let labelTasks = lead.string.MyLeads

  let search = ''
  const currentUser = getCurrentAccount() as EmployeeAccount
  const assigned = { assignee: currentUser.employee }
  const created = { createdBy: currentUser._id }
  let subscribed = { _id: { $in: [] as Ref<Task>[] } }

  $: baseQuery = updateBaseQuery(mode, { assigned, created, subscribed })
  function updateBaseQuery (mode: string, queries: { [key: string]: DocumentQuery<Lead> }) {
    return { ...queries[mode] }
  }
  let searchQuery: DocumentQuery<Lead> = { ...baseQuery }
  function updateSearchQuery (search: string): void {
    searchQuery = search === '' ? { ...baseQuery } : { ...baseQuery, $search: search }
  }
  $: if (baseQuery) updateSearchQuery(search)
  $: resultQuery = { ...searchQuery }

  const subscribedQuery = createQuery()
  function getSubscribed () {
    subscribedQuery.query(
      _class,
      { 'notification:mixin:Collaborators.collaborators': getCurrentAccount()._id },
      (result) => {
        const newSub = result.map((p) => p._id as Ref<AttachedDoc> as Ref<Lead>)
        const curSub = subscribed._id.$in
        if (curSub.length !== newSub.length || curSub.some((id, i) => newSub[i] !== id)) {
          subscribed = { _id: { $in: newSub } }
        }
      },
      { sort: { _id: 1 } }
    )
  }
  $: if (mode === 'subscribed') getSubscribed()
  const config: [string, IntlString, object][] = [
    ['assigned', view.string.Assigned, {}],
    ['created', view.string.Created, {}],
    ['subscribed', view.string.Subscribed, {}]
  ]
  let [[mode]] = config
  function handleChangeMode (newMode: string) {
    if (newMode === mode) return
    mode = newMode
  }
  $: modeSelectorProps = {
    config,
    mode,
    onChange: handleChangeMode
  } as IModeSelector

  let viewlet: Viewlet | undefined
  let loading = true

  let key = makeViewletKey()
  let preference: ViewletPreference | undefined
  onDestroy(
    resolvedLocationStore.subscribe((loc) => {
      key = makeViewletKey(loc)
    })
  )

  const preferenceQuery = createQuery()
  const client = getClient()
  client
    .findOne<Viewlet>(view.class.Viewlet, { attachTo: _class, descriptor: task.viewlet.StatusTable })
    .then((res) => {
      viewlet = res
      preferenceQuery.query(
        view.class.ViewletPreference,
        {
          attachedTo: res._id
        },
        (res) => {
          preference = res[0]
          loading = false
        },
        { limit: 1 }
      )
    })
  $: viewOptions = getViewOptions(viewlet, $viewOptionStore)
</script>

<div
  class="ac-header full divide"
  class:header-with-mode-selector={modeSelectorProps !== undefined}
  class:header-without-label={!labelTasks}
>
  <div class="ac-header__wrap-title">
    <span class="ac-header__title"><Label label={labelTasks} /></span>
    {#if modeSelectorProps !== undefined}
      <ModeSelector props={modeSelectorProps} />
    {/if}
  </div>
</div>
<div class="ac-header full divide search-start">
  <div class="ac-header-full small-gap">
    <SearchEdit bind:value={search} />
    <div class="buttons-divider" />
    <FilterButton {_class} />
  </div>
  {#if viewlet}
    <ViewletSettingButton bind:viewOptions {viewlet} />
  {/if}
</div>
<FilterBar {_class} query={searchQuery} {viewOptions} on:change={(e) => (resultQuery = e.detail)} />

{#if viewlet}
  {#if loading}
    <Loading />
  {:else}
    <TableBrowser
      {_class}
      config={preference?.config ?? viewlet.config}
      options={viewlet.options}
      query={resultQuery}
      showNotification
    />
  {/if}
{/if}