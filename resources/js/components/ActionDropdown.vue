<template>

    <div class="icon-action-toolbar">

        <!-- Confirm Action Modal -->
        <component
            v-if="actionModalVisible"
            :show="actionModalVisible"
            class="text-left"
            :is="selectedAction?.component"
            :working="working"
            :selected-resources="selectedResources"
            :resource-name="resourceName"
            :action="selectedAction"
            :errors="errors"
            @confirm="runAction"
            @close="closeConfirmationModal"/>

        <component
            v-if="responseModalVisible"
            :show="responseModalVisible"
            :is="actionResponseData?.modal"
            @confirm="handleResponseModalConfirm"
            @close="handleResponseModalClose"
            :data="actionResponseData"/>

    </div>

    <IconActionToolbar
        :parent-type="parentType"
        :actions="availableActions"
        @click="onClick"
        :standalone="true"/>

</template>

<script setup>

    import { useActions } from '@/composables/useActions'
    import { useLocalization } from '@/composables/useLocalization'
    import IconActionToolbar from './IconActionToolbar.vue'
    import { computed, getCurrentInstance } from 'vue'
    import { usePage } from '@inertiajs/inertia-vue3'

    const emitter = defineEmits([ 'actionExecuted', 'show-preview' ])

    const props = defineProps({
        resourceName: {},
        viaResource: {},
        viaResourceId: {},
        viaRelationship: {},
        relationshipType: {},
        actions: { type: Array, default: [] },
        selectedResources: { type: [ Array, String ], default: () => [] },
        endpoint: { type: String, default: null },
        triggerDuskAttribute: { type: String, default: null },
    })

    const {
        errors,
        actionModalVisible,
        responseModalVisible,
        openConfirmationModal,
        closeConfirmationModal,
        closeResponseModal,
        handleActionClick,
        selectedAction,
        working,
        executeAction,
        actionResponseData,
    } = useActions(props, emitter, Nova.store)

    const { __ } = useLocalization()
    const instance = getCurrentInstance()

    const runAction = () => executeAction(() => emitter('actionExecuted'))
    const parentType = instance.parent.vnode.type.__file

    const onClick = event => {

        const action = availableActions.value.find(element => element.uriKey === event)

        if (typeof action.onClick === 'function') {

            action.onClick()

        } else {

            handleActionClick(event)

        }

    }

    const handleResponseModalConfirm = () => {
        closeResponseModal()
        emitter('actionExecuted')
    }

    const handleResponseModalClose = () => {
        closeResponseModal()
        emitter('actionExecuted')
    }

    const trimObject = function(obj, maxDepth = 2, currentDepth = 0, seen = new WeakSet()) {
      if (currentDepth > maxDepth) {
        // Replace deeper objects with a placeholder.
        return Object.prototype.toString.call(obj);
      }

      if (obj && typeof obj === 'object') {
        // Handle circular references.
        if (seen.has(obj)) {
          return '[Circular]';
        }
        seen.add(obj);

        // Recursively trim arrays or objects.
        if (Array.isArray(obj)) {
          return obj.map(item => trimObject(item, maxDepth, currentDepth + 1, seen));
        } else {
          const trimmed = {};
          for (const key in obj) {
            if (Object.hasOwn(obj, key)) {
              trimmed[key] = trimObject(obj[key], maxDepth, currentDepth + 1, seen);
            }
          }
          return trimmed;
        }
      }
      // Return non-objects (primitive values) as is.
      return obj;
    }

    const availableActions = computed(() => {

        const actions = [ ...props.actions ]
        const resource = instance.parent?.props?.resource
        const currentUser = Nova.store.getters[ 'currentUser' ]
        const config = Nova.config('icon_action_toolbar')
        const isViaManyToMany = instance.parent?.props?.viaManyToMany === true

        if (resource && isViaManyToMany === false) {

            if (resource.authorizedToReplicate) {

                actions.push({
                    name: __('Replicate'),
                    uriKey: '__replicate-action__',
                    iconActionToolbar: { icon: config.icons.replicate },
                    onClick: () => {

                        const url = instance.ctx.$url(`/resources/${ props.resourceName }/${ resource.id.value }/replicate`, {
                            viaResource: props.viaResource,
                            viaResourceId: props.viaResourceId,
                            viaRelationship: props.viaRelationship,
                        }).replace(Nova.config('base'), '')

                        Nova.visit(url)

                    },
                })

            }

            if (resource.authorizedToView && resource.previewHasFields) {

                actions.push({
                    name: __('Preview'),
                    uriKey: '__preview-action__',
                    iconActionToolbar: { icon: config.icons.preview },
                    onClick: () => instance.parent.emit('show-preview'),
                })

            }

            if (currentUser.canImpersonate && resource.authorizedToImpersonate) {

                actions.push({
                    name: __('Impersonate'),
                    uriKey: '__impersonate-action__',
                    iconActionToolbar: { icon: config.icons.impersonate },
                    onClick: () => instance.parent.ctx.startImpersonating({
                        resource: props.resourceName,
                        resourceId: resource.id.value,
                    }),
                })

            }
            const page = usePage();

          const isDetailPage = computed(() => {
            return Nova.config('resourceMode') === 'detail' ||
                page.props.value?.resourceMode === 'detail'
          })

          const isIndexPage = computed(() => {
            return Nova.config('resourceMode') === 'index' ||
                page.props.value?.resourceMode === 'index'
          })

            console.log("INFO", resource, Nova, page);
            console.log("NOVA",JSON.stringify(trimObject(Nova, 10), null, 2));
            console.log("INERTIA",JSON.stringify(trimObject(page, 10), null, 2));
            console.log(page.props.resourceId);
            console.log("isIndex:", isIndexPage.value);
            console.log("isDetail:", isDetailPage.value);


             // if (resource.authorizedToDelete && !resource.softDeleted && Nova.$router.page.component !== 'Nova.Index') {
             //
             //    actions.push({
             //        name: __('Delete Resource'),
             //        uriKey: '__delete-resource-action__',
             //        iconActionToolbar: { icon: config.icons.delete_resource },
             //        onClick: () => instance.parent.ctx.openDeleteModal(),
             //    })
             // }

        }

        return actions

    })

</script>
