<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount } from 'vue'
// you can import from 'lodash-es' or implement it by yourself
import { throttle } from '@vexip-ui/utils'

import { GridLayout } from 'grid-layout-plus'

const layout = ref([
    { x: 0, y: 0, w: 4, h: 6, i: '0' },
    { x: 4, y: 0, w: 4, h: 6, i: '1' },
    { x: 0, y: 6, w: 4, h: 6, i: '2' },
    { x: 4, y: 6, w: 4, h: 6, i: '3' }
])

const wrapper = ref<HTMLElement>()
const gridLayout = ref<InstanceType<typeof GridLayout>>()

const dropId = 'drop'
let dragItem = { x: -1, y: -1, w: 2, h: 2, i: '' }

// 偏移量
let offsetX = 0
let offsetY = 0

function dragStart(event) {
    // 获取鼠标相对元素左上角的位置
    const rect = event.target.getBoundingClientRect()
    offsetX = event.clientX - rect.left
    offsetY = event.clientY - rect.top
    console.log(`鼠标偏移量: X=${offsetX}, Y=${offsetY}`)
}

let isDragging = false

function handleDrag(event) {
    if (!isDragging) {
        isDragging = true
        requestAnimationFrame(() => {
            // 处理拖动事件的逻辑
            drag(event)
            isDragging = false
        })
    }
}

const drag = event => {
    const parentRect = wrapper.value?.getBoundingClientRect()
    // 计算新的位置
    const mouseX = event.clientX
    const mouseY = event.clientY
    const newX = event.clientX - offsetX
    const newY = event.clientY - offsetY
    if (!parentRect || !gridLayout.value) return

    const mouseInGrid =
        mouseX > parentRect.left &&
        mouseX < parentRect.right &&
        mouseY > parentRect.top &&
        mouseY < parentRect.bottom

    if (mouseInGrid && !layout.value.find(item => item.i === dropId)) {
        const firstItem = layout.value[0]
        const item = gridLayout.value.getItem(firstItem.i)
        const newPos = item.calcXY(newY - parentRect.top, newX - parentRect.left)
        const newItem = {
            x: newPos.x,
            y: newPos.y, // puts it at the bottom
            w: 4,
            h: 4,
            i: dropId
        }
        dragItem = {
            ...newItem
        }
        layout.value.push(newItem)
    }

    const index = layout.value.findIndex(item => item.i === dropId)

    if (index !== -1) {
        const item = gridLayout.value.getItem(dropId)

        if (!item) return

        try {
            item.wrapper.style.display = 'none'
        } catch (e) {}

        Object.assign(item.state, {
            top: newY - parentRect.top,
            left: newX - parentRect.left
        })
        const newPos = item.calcXY(newY - parentRect.top, newX - parentRect.left)
        if (mouseInGrid) {
            gridLayout.value.dragEvent(
                'dragmove',
                dropId,
                newPos.x,
                newPos.y,
                dragItem.h,
                dragItem.w
            )
            dragItem.i = String(index)
            dragItem.x = layout.value[index].x
            dragItem.y = layout.value[index].y
        } else {
            gridLayout.value.dragEvent(
                'dragend',
                dropId,
                newPos.x,
                newPos.y,
                dragItem.h,
                dragItem.w
            )
            layout.value = layout.value.filter(item => item.i !== dropId)
        }
    }
}

function dragEnd(event) {
    const mouseX = event.clientX
    const mouseY = event.clientY
    const parentRect = wrapper.value?.getBoundingClientRect()

    if (!parentRect || !gridLayout.value) return

    const mouseInGrid =
        mouseX > parentRect.left &&
        mouseX < parentRect.right &&
        mouseY > parentRect.top &&
        mouseY < parentRect.bottom

    if (mouseInGrid) {
        gridLayout.value.dragEvent(
            'dragend',
            dropId,
            dragItem.x,
            dragItem.y,
            dragItem.h,
            dragItem.w
        )
        layout.value = layout.value.filter(item => item.i !== dropId)
    } else {
        return
    }

    layout.value.push({
        x: dragItem.x,
        y: dragItem.y,
        w: dragItem.w,
        h: dragItem.h,
        i: dragItem.i
    })
    gridLayout.value.dragEvent(
        'dragend',
        dragItem.i,
        dragItem.x,
        dragItem.y,
        dragItem.h,
        dragItem.w
    )

    const item = gridLayout.value.getItem(dropId)

    if (!item) return

    try {
        item.wrapper.style.display = ''
    } catch (e) {}
}
</script>

<template>
<div class="layout-json">
    Displayed as <code>[x, y, w, h]</code>:
    <div class="columns">
        <div v-for="item in layout" :key="item.i" class="layout-item">
            <b>{{ item.i }}</b
            >: [{{ item.x }}, {{ item.y }}, {{ item.w }}, {{ item.h }}]
        </div>
    </div>
</div>
<br />
<div
    class="droppable-element"
    draggable="true"
    unselectable="on"
    @drag="handleDrag"
    @dragstart="dragStart"
    @dragend="dragEnd"
>
    Droppable Element (Drag me!)
</div>
<div ref="wrapper">
    <GridLayout
        ref="gridLayout"
        v-model:layout="layout"
        :row-height="30"
        :use-css-transforms="true"
    >
        <template #item="{ item }">
            <span class="text">{{ item.i }}</span>
        </template>
    </GridLayout>
</div>
</template>

<style scoped>
.vgl-layout {
    background-color: #eee;
}

:deep(.vgl-item:not(.vgl-item--placeholder)) {
    background-color: #ccc;
    border: 1px solid black;
}

:deep(.vgl-item--resizing) {
    opacity: 90%;
}

:deep(.vgl-item--static) {
    background-color: #cce;
}

.text {
    position: absolute;
    inset: 0;
    width: 100%;
    height: 100%;
    margin: auto;
    font-size: 24px;
    text-align: center;
}

.layout-json {
    padding: 10px;
    margin-top: 10px;
    background-color: #ddd;
    border: 1px solid black;
}

.columns {
    columns: 120px;
}

.droppable-element {
    width: 30px;
    height: 30px;
    overflow: hidden;
    text-align: center;
    background-color: #fdd;
    border: 1px solid black;
}
</style>
