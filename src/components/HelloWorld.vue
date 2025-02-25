<script lang="ts" setup>
import { ref } from 'vue';
import draggable from 'vuedraggable';

interface TreeItem {
  id: number;
  name: string;
  type: 'folder' | 'file';
  children?: TreeItem[];
}

const selected = ref<number[]>([]);

const treeData = ref<TreeItem[]>([
  {
    id: 1,
    name: 'Main Folder',
    type: 'folder',
    children: [
      { id: 2, name: 'File 1', type: 'file' },
      { id: 3, name: 'Sub Folder', type: 'folder', children: [] }
    ]
  }
]);

const editingItem = ref<TreeItem | null>(null);

const getParentArray = (item: TreeItem): TreeItem[] => {
  const findParent = (items: TreeItem[], targetId: number): TreeItem[] | null => {
    for (const i of items) {
      if (i.id === targetId) return items;
      if (i.children) {
        const found = findParent(i.children, targetId);
        if (found) return found;
      }
    }
    return null;
  };

  return findParent(treeData.value, item.id) || treeData.value;
};

const addItem = (parent: TreeItem) => {
  if (!parent.children) parent.children = [];
  const newItem: TreeItem = {
    id: Date.now(),
    name: 'New Item',
    type: parent.type === 'folder' ? 'file' : 'folder',
  };
  parent.children.push(newItem);
};

const deleteItem = (item: TreeItem) => {
  if (item.id === 1) {
    // Prevent deletion of the main folder
    console.log('Main folder cannot be deleted');
    return;
  }

  const deleteRecursive = (items: TreeItem[], id: number): boolean => {
    const index = items.findIndex(i => i.id === id);
    if (index !== -1) {
      items.splice(index, 1);
      return true;
    }
    for (const i of items) {
      if (i.children && deleteRecursive(i.children, id)) {
        return true;
      }
    }
    return false;
  };

  // To delete the item from the root level or sub-level
  if (!deleteRecursive(treeData.value, item.id)) {
    console.error('Item not found');
  }
};

const startEditing = (item: TreeItem) => {
  editingItem.value = { ...item };
};

const finishEditing = () => {
  if (editingItem.value) {
    const updateName = (items: TreeItem[]) => {
      for (const i of items) {
        if (i.id === editingItem.value!.id) {
          i.name = editingItem.value!.name;
        } else if (i.children) {
          updateName(i.children);
        }
      }
    };
    updateName(treeData.value);
    editingItem.value = null;
  }
};

const onDragChange = (event: any, item: TreeItem) => {
  if (event.added) {
    const newIndex = event.added.newIndex;
    const parentArray = getParentArray(item);
    const movedItem = parentArray[newIndex];

    if (event.added.element && event.added.element.id !== item.id) {
      const oldParentArray = getParentArray(event.added.element);
      const oldIndex = oldParentArray.findIndex(i => i.id === event.added.element.id);
      oldParentArray.splice(oldIndex, 1);
    }
  }
};

const dialogVisible = ref(false);
const newItemName = ref('');
const currentParentItem = ref<TreeItem | null>(null);
const itemType = ref<'category' | 'table'>('category');


const nameRequiredRule = (value: string) => !!value || 'Name is required';


const openAddItemDialog = (type: 'category' | 'table', parent: TreeItem) => {
  itemType.value = type;
  currentParentItem.value = parent;
  newItemName.value = '';
  dialogVisible.value = true;
};


const createItem = () => {
  if (!newItemName.value) return;

  const newItem: TreeItem = {
    id: Date.now(),
    name: newItemName.value,
    type: itemType.value === 'category' ? 'folder' : 'file',
  };

  if (!currentParentItem.value?.children) currentParentItem.value.children = [];
  currentParentItem.value.children.push(newItem);

  dialogVisible.value = false; 
};
</script>

<template>
  <v-treeview
    v-model="selected"
    :items="treeData"
    item-key="id"
    open-on-click
    activatable
    dense
  >
    <template #prepend="{ item }">
      <v-icon v-if="item.type === 'folder'">mdi-folder</v-icon>
      <v-icon v-else>mdi-file</v-icon>
    </template>

    <template #append="{ item }">
      <v-menu bottom left>
        <template #activator="{ props }" >
          <v-btn icon small v-bind="props">
            <v-icon color="blue">mdi-plus</v-icon>
          </v-btn>
          <v-btn
            v-if="item.id !== 1"
            icon
            small
            @click.stop="deleteItem(item)"
          >
            <v-icon color="red">mdi-delete</v-icon>
          </v-btn>
        </template>
        <v-list>
          <v-list-item @click="openAddItemDialog('category', item)">
            <v-list-item-title>Create Category</v-list-item-title>
          </v-list-item>
          <v-list-item @click="openAddItemDialog('table', item)">
            <v-list-item-title>Create Table</v-list-item-title>
          </v-list-item>
        </v-list>
      </v-menu>
    </template>

    <template #title="{ item }">
      <div @dblclick="startEditing(item)">
        <template v-if="editingItem?.id === item.id">
          <v-text-field
            v-model="editingItem.name"
            dense
            solo
            hide-details
            @blur="finishEditing"
            @keyup.enter="finishEditing"
          />
        </template>
        <template v-else>
          <span>{{ item.name }}</span>
          <!-- Delete Button (except Main Folder) -->

        </template>
      </div>
    </template>

    <template #label="{ item }">
      <draggable
        :list="getParentArray(item)"
        group="tree"
        item-key="id"
        handle=".handle"
        @change="onDragChange($event, item)"
      >
        <template #item="{ element }">
          <div class="handle" style="cursor: move">
            {{ element.name }}
          </div>
        </template>
      </draggable>
    </template>
  </v-treeview>

  <!-- Dialog for Naming the New Category/Table -->
  <v-dialog v-model="dialogVisible" max-width="400px">
    <v-card>
      <v-card-title class="headline">Enter Name</v-card-title>
      <v-card-text>
        <v-text-field
          v-model="newItemName"
          label="Name"
          :rules="[nameRequiredRule]"
        />
      </v-card-text>
      <v-card-actions>
        <v-btn text @click="dialogVisible = false">Cancel</v-btn>
        <v-btn color="primary" @click="createItem">Create</v-btn>
      </v-card-actions>
    </v-card>
  </v-dialog>
</template>

<style scoped>
.handle {
  cursor: move;
}
</style>
