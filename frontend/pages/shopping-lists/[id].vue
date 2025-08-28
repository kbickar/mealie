<!-- Recipe References -->      <!-- Reorder Labels -->            <!-- Checked Items for this label -->            <!-- Unchecked Items --><template>
  <v-container
    v-if="shoppingList"
    class="md-container"
  >
    <BaseDialog
      v-model="state.checkAllDialog"
      :title="$t('general.confirm')"
      can-confirm
      @confirm="checkAll"
    >
      <v-card-text>
        {{ $t('shopping-list.are-you-sure-you-want-to-check-all-items') }}
      </v-card-text>
    </BaseDialog>

    <BaseDialog
      v-model="state.uncheckAllDialog"
      :title="$t('general.confirm')"
      can-confirm
      @confirm="uncheckAll"
    >
      <v-card-text>
        {{ $t('shopping-list.are-you-sure-you-want-to-uncheck-all-items') }}
      </v-card-text>
    </BaseDialog>

    <BaseDialog
      v-model="state.deleteCheckedDialog"
      :title="$t('general.confirm')"
      can-confirm
      @confirm="deleteChecked"
    >
      <v-card-text>
        {{ $t('shopping-list.are-you-sure-you-want-to-delete-checked-items') }}
      </v-card-text>
    </BaseDialog>

    <BasePageTitle divider>
      <template #header>
        <v-container>
          <v-row>
            <v-col
              class="text-left"
            >
              <ButtonLink
                :to="`/shopping-lists?disableRedirect=true`"
                :text="$t('shopping-list.all-lists')"
                :icon="$globals.icons.backArrow"
              />
            </v-col>
            <v-col
              v-if="mdAndUp"
              cols="6"
              class="d-none d-sm-flex justify-center"
            >
              <v-img
                max-height="100"
                max-width="100"
                :src="require('~/static/svgs/shopping-cart.svg')"
              />
            </v-col>
            <v-col class="d-flex justify-end">
              <BaseButtonGroup
                :buttons="[
                  {
                    icon: $globals.icons.contentCopy,
                    text: '',
                    event: 'edit',
                    children: [
                      {
                        icon: $globals.icons.contentCopy,
                        text: $t('shopping-list.copy-as-text'),
                        event: 'copy-plain',
                      },
                      {
                        icon: $globals.icons.contentCopy,
                        text: $t('shopping-list.copy-as-markdown'),
                        event: 'copy-markdown',
                      },
                    ],
                  },
                  {
                    icon: $globals.icons.checkboxOutline,
                    text: $t('shopping-list.check-all-items'),
                    event: 'check',
                  },
                  {
                    icon: $globals.icons.dotsVertical,
                    text: '',
                    event: 'three-dot',
                    children: [
                      {
                        icon: $globals.icons.tags,
                        text: $t('shopping-list.reorder-labels'),
                        event: 'reorder-labels',
                      },
                      {
                        icon: $globals.icons.tags,
                        text: $t('shopping-list.manage-labels'),
                        event: 'manage-labels',
                      },
                    ],
                  },
                ]"
                @edit="edit = true"
                @three-dot="threeDot = true"
                @check="openCheckAll"
                @copy-plain="copyListItems('plain')"
                @copy-markdown="copyListItems('markdown')"
                @reorder-labels="toggleReorderLabelsDialog()"
                @manage-labels="$router.push(`/group/data/labels`)"
              />
            </v-col>
          </v-row>
        </v-container>
      </template>
      <template #title>
        {{ shoppingList.name }}
      </template>
    </BasePageTitle>
    <BannerWarning
      v-if="isOffline"
      :title="$t('shopping-list.you-are-offline')"
      :description="$t('shopping-list.you-are-offline-description')"
    />

    <!-- Viewer -->
    <section
      v-if="!edit"
      class="py-2"
    >
      <!-- Create Item -->
      <div v-if="createEditorOpen">
        <ShoppingListItemEditor
          v-model="createListItemData"
          class="my-4"
          :labels="allLabels || []"
          :units="allUnits || []"
          :foods="allFoods || []"
          :allow-delete="false"
          @delete="createEditorOpen = false"
          @cancel="createEditorOpen = false"
          @save="createListItem"
        />
      </div>
      <div v-else class="d-flex justify-end">
        <BaseButton
          create
          @click="createEditorOpen = true"
        >
          {{ $t('general.add') }}
        </BaseButton>
      </div>

      <div
        v-for="labelName in allVisibleLabels"
        :key="labelName"
        class="pb-4"
      >
        <v-btn
          :color="getLabelColor(itemsByLabel[labelName]?.[0] || checkedItemsByLabel[labelName]?.[0]) ? getLabelColor(itemsByLabel[labelName]?.[0] || checkedItemsByLabel[labelName]?.[0]) : '#959595'"
          :style="{
            'color': getTextColor(getLabelColor(itemsByLabel[labelName]?.[0] || checkedItemsByLabel[labelName]?.[0])),
            'letter-spacing': 'normal',
          }"
          @click="toggleShowLabel(labelName.toString())"
        >
          <v-icon>
            {{ labelOpenState[labelName] ? $globals.icons.chevronDown : $globals.icons.chevronRight }}
          </v-icon>
          {{ labelName }}
        </v-btn>
        <v-divider />
        <v-expand-transition>
          <div v-if="labelOpenState[labelName]">
            <VueDraggable
              v-if="itemsByLabel[labelName] && itemsByLabel[labelName].length > 0"
              :model-value="itemsByLabel[labelName]"
              handle=".handle"
              :delay="250"
              :delay-on-touch-only="true"
              @start="loadingCounter += 1"
              @end="loadingCounter -= 1"
              @update:model-value="updateIndexUncheckedByLabel(labelName.toString(), $event)"
            >
              <v-lazy
                v-for="(item, index) in itemsByLabel[labelName]"
                :key="item.id"
                class="ml-2 my-2"
              >
                <ShoppingListItem
                  v-model="itemsByLabel[labelName][index]"
                  :labels="allLabels || []"
                  :units="allUnits || []"
                  :foods="allFoods || []"
                  :recipes="recipeMap"
                  @checked="saveListItem"
                  @save="saveListItem"
                  @delete="deleteListItem(item)"
                />
              </v-lazy>
            </VueDraggable>

            <div
              v-if="showCheckedItems && checkedItemsByLabel[labelName] && checkedItemsByLabel[labelName].length > 0"
              class="mt-4 ml-2"
            >
              <div class="d-flex align-center mb-2">
                <button @click="toggleShowCheckedForLabel(labelName.toString())" class="d-flex align-center">
                  <v-icon class="mr-1" size="small">
                    {{ checkedLabelOpenState[labelName] ? $globals.icons.chevronDown : $globals.icons.chevronRight }}
                  </v-icon>
                  <span class="text-caption text-medium-emphasis">
                    {{ $t('shopping-list.items-checked-count', checkedItemsByLabel[labelName].length) }}
                  </span>
                </button>
                <v-spacer />
                <div class="d-flex">
                  <v-btn
                    icon
                    size="small"
                    variant="text"
                    @click="uncheckAllForLabel(labelName.toString())"
                    :title="$t('shopping-list.uncheck-all-items')"
                  >
                    <v-icon size="small">
                      {{ $globals.icons.checkboxBlankOutline }}
                    </v-icon>
                  </v-btn>
                  <v-btn
                    icon
                    size="small"
                    variant="text"
                    @click="deleteCheckedForLabel(labelName.toString())"
                    :title="$t('shopping-list.delete-checked')"
                  >
                    <v-icon size="small">
                      {{ $globals.icons.delete }}
                    </v-icon>
                  </v-btn>
                </div>
              </div>
              <v-expand-transition>
                <div v-if="checkedLabelOpenState[labelName]">
                  <div
                    v-for="(item, idx) in checkedItemsByLabel[labelName]"
                    :key="item.id"
                    class="my-1"
                  >
                    <ShoppingListItem
                      v-model="checkedItemsByLabel[labelName][idx]"
                      class="strike-through-note"
                      :labels="allLabels || []"
                      :units="allUnits || []"
                      :foods="allFoods || []"
                      :hide-completed-summary="!showCheckedSummary"
                      @checked="saveListItem"
                      @save="saveListItem"
                      @delete="deleteListItem(item)"
                    />
                  </div>
                </div>
              </v-expand-transition>
            </div>
          </div>
        </v-expand-transition>
      </div>

      <BaseDialog
        v-model="reorderLabelsDialog"
        :icon="$globals.icons.tagArrowUp"
        :title="$t('shopping-list.reorder-labels')"
        :submit-icon="$globals.icons.save"
        :submit-text="$t('general.save')"
        can-submit
        @submit="saveLabelOrder"
        @close="cancelLabelOrder"
      >
        <v-card
          height="fit-content"
          max-height="70vh"
          style="overflow-y: auto;"
        >
          <VueDraggable
            v-if="localLabels"
            v-model="localLabels"
            handle=".handle"
            :delay="250"
            :delay-on-touch-only="true"
            class="my-2"
            @update:model-value="updateLabelOrder"
          >
            <div
              v-for="(labelSetting, index) in localLabels"
              :key="labelSetting.id"
            >
              <MultiPurposeLabelSection
                v-model="localLabels[index]"
                use-color
              />
            </div>
          </VueDraggable>
        </v-card>
      </BaseDialog>
    </section>

    <v-lazy
      v-if="shoppingList.recipeReferences && shoppingList.recipeReferences.length > 0"
    >
      <section>
        <div>
          <span>
            <v-icon start class="mb-1">
              {{ $globals.icons.primary }}
            </v-icon>
          </span>
          {{ $t('shopping-list.linked-recipes-count', shoppingList.recipeReferences
            ? shoppingList.recipeReferences.length
            : 0) }}
        </div>
        <v-divider class="my-4" />
        <RecipeList
          :recipes="recipeList"
          show-description
          :disabled="isOffline"
        >
          <template
            v-for="(recipe, index) in recipeList"
            #[`actions-${recipe.id}`]
            :key="'item-actions-decrease' + recipe.id"
          >
            <v-list-item-action>
              <v-btn
                v-if="recipe"
                icon
                flat
                class="bg-transparent"
                :disabled="isOffline"
                @click.prevent="removeRecipeReferenceToList(recipe.id!)"
              >
                <v-icon color="grey-lighten-1">
                  {{ $globals.icons.minus }}
                </v-icon>
              </v-btn>
            </v-list-item-action>
            <div class="pl-3">
              {{ shoppingList.recipeReferences[index].recipeQuantity }}
            </div>
            <v-list-item-action>
              <v-btn
                icon
                :disabled="isOffline"
                flat
                class="bg-transparent"
                @click.prevent="addRecipeReferenceToList(recipe.id!)"
              >
                <v-icon color="grey-lighten-1">
                  {{ $globals.icons.createAlt }}
                </v-icon>
              </v-btn>
            </v-list-item-action>
          </template>
        </RecipeList>
      </section>
    </v-lazy>
    <WakelockSwitch />
    <div
      class="d-print-none d-flex px-2"
      :class="$vuetify.display.smAndDown ? 'justify-center' : 'justify-end'"
    >
      <v-switch
        v-model="showCheckedItems"
        color="primary"
        :label="$t('shopping-list.show-checked-items')"
      />
    </div>
    <div
      v-if="showCheckedItems"
      class="d-print-none d-flex px-2"
      :class="$vuetify.display.smAndDown ? 'justify-center' : 'justify-end'"
    >
      <v-switch
        v-model="showCheckedSummary"
        color="primary"
        :label="$t('shopping-list.show-completion-dates')"
      />
    </div>
  </v-container>
</template>

<script lang="ts">
import { VueDraggable } from "vue-draggable-plus";
import MultiPurposeLabelSection from "~/components/Domain/ShoppingList/MultiPurposeLabelSection.vue";
import ShoppingListItem from "~/components/Domain/ShoppingList/ShoppingListItem.vue";
import RecipeList from "~/components/Domain/Recipe/RecipeList.vue";
import ShoppingListItemEditor from "~/components/Domain/ShoppingList/ShoppingListItemEditor.vue";
import { useFoodStore, useLabelStore, useUnitStore } from "~/composables/store";
import { useShoppingListPreferences } from "~/composables/use-users/preferences";
import { getTextColor } from "~/composables/use-text-color";
import { useShoppingListPage } from "~/composables/shopping-list-page/use-shopping-list-page";

export default defineNuxtComponent({
  components: {
    VueDraggable,
    MultiPurposeLabelSection,
    ShoppingListItem,
    RecipeList,
    ShoppingListItemEditor,
  },
  setup() {
    const { mdAndUp } = useDisplay();
    const i18n = useI18n();
    const $auth = useMealieAuth();
    const preferences = useShoppingListPreferences();

    useSeoMeta({
      title: i18n.t("shopping-list.shopping-list"),
    });

    const route = useRoute();
    const groupSlug = computed(() => route.params.groupSlug as string || $auth.user.value?.groupSlug || "");
    const id = route.params.id as string;

    const shoppingListPage = useShoppingListPage(id);
    const { store: allLabels } = useLabelStore();
    const { store: allUnits } = useUnitStore();
    const { store: allFoods } = useFoodStore();

    const checkedLabelOpenState = ref<{[key: string]: boolean}>({});
    const showCheckedItems = ref(true);
    const showCheckedSummary = ref(true);

    const checkedItemsByLabel = computed(() => {
      if (!shoppingListPage.listItems?.checked) {
        return {};
      }

      const grouped: {[key: string]: any[]} = {};

      shoppingListPage.listItems.checked.forEach(item => {
        const labelName = item.label?.name || 'No Label';
        if (!grouped[labelName]) {
          grouped[labelName] = [];
        }
        grouped[labelName].push(item);
      });

      return grouped;
    });

    const allVisibleLabels = computed(() => {
      const labels = new Set<string>();

      if (shoppingListPage.itemsByLabel?.value) {
        Object.keys(shoppingListPage.itemsByLabel.value).forEach(label => {
          labels.add(label);
        });
      }

      if (showCheckedItems.value && checkedItemsByLabel.value) {
        Object.keys(checkedItemsByLabel.value).forEach(label => {
          labels.add(label);
        });
      }

      return Array.from(labels).sort();
    });

    function toggleShowCheckedForLabel(labelName: string) {
      checkedLabelOpenState.value[labelName] = !checkedLabelOpenState.value[labelName];
    }

    async function uncheckAllForLabel(labelName: string) {
      const itemsToUncheck = checkedItemsByLabel.value[labelName];
      if (!itemsToUncheck) return;

      for (const item of itemsToUncheck) {
        item.checked = false;
        await shoppingListPage.saveListItem(item);
      }
    }

    async function deleteCheckedForLabel(labelName: string) {
      const itemsToDelete = checkedItemsByLabel.value[labelName];
      if (!itemsToDelete) return;

      for (const item of itemsToDelete) {
        await shoppingListPage.deleteListItem(item);
      }
    }

    return {
      groupSlug,
      preferences,
      allLabels,
      allUnits,
      allFoods,
      getTextColor,
      mdAndUp,
      checkedLabelOpenState,
      checkedItemsByLabel,
      allVisibleLabels,
      showCheckedItems,
      showCheckedSummary,
      toggleShowCheckedForLabel,
      uncheckAllForLabel,
      deleteCheckedForLabel,
      ...shoppingListPage,
    };
  },
});
</script>

<style scoped>
.number-input-container {
  max-width: 50px;
}

.strike-through-note {
  opacity: 0.7;
}
</style>
