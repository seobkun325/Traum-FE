<template>
  <div>
    <button
      class="text-xl active:bg-emerald-600 font-bold uppercase px-6 py-3 rounded hover:shadow-lg outline-none focus:outline-none mr-1 mb-1 ease-linear transition-all duration-150"
      type="button"
      v-on:click="toggleModal()"
    >
      <i class="fas fa-cog mr-2"></i>
    </button>
    <div v-if="showModal" class="fixed inset-0 z-50 flex items-center justify-center">
      <div>
        <!--content 모달창의 크기조절 위치-->
        <div class="text-white bg-gray-600">
          <!--header-->
          <div class="flex items-center justify-between p-5">
            <h3 class="text-3xl font-semibold text-center w-full">설정 변경</h3>
            <button
              class="p-1 ml-auto bg-transparent border-0 text-black opacity-5 float-right text-3xl leading-none font-semibold outline-none focus:outline-none"
              v-on:click="toggleModal()"
            >
              <span
                class="bg-transparent text-black opacity-5 h-6 w-6 text-2xl block outline-none focus:outline-none"
              >
                ×
              </span>
            </button>
          </div>
          <!--body-->
          <div>
            <p class="my-4 text-blueGray-500 text-lg leading-relaxed"></p>
            <div class="flex flex-col">
              <canvas
                ref="bjsCanvas"
                class="mx-auto w-[95%] mb-4"
                @dblclick="item?.meshId == 1 ? routeToUnity() : null"
              ></canvas>
              <div class="space-y-4 mx-auto w-[95%] text-black">
                <div class="flex items-center justify-between">
                  <label for="name" class="block text-sm font-medium text-white">Name</label>
                  <input
                    id="name"
                    v-model="localItem['name']"
                    class="ml-16 mt-1 block w-[41%] px-3 py-2 bg-white border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-indigo-500 focus:border-indigo-500 sm:text-sm"
                  />
                </div>
                <div class="flex justify-between items-center">
                  <label for="threshold" class="block text-sm font-medium text-white"
                    >Threshold</label
                  >
                  <div>
                    <div class="mt-1 flex items-center justify-end space-x-2">
                      <input
                        id="threshold"
                        v-model="localItem['threshold']"
                        type="number"
                        class="block w-[40%] px-3 py-2 bg-white border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-indigo-500 focus:border-indigo-500 sm:text-sm"
                      />
                      <fwb-range
                        v-model="localItem['threshold']"
                        label=""
                        size="sm"
                        :steps="1"
                        :max="100"
                        :min="0"
                        class="flex-grow w-1"
                      />
                    </div>
                  </div>
                </div>
                <div class="flex justify-between items-center">
                  <div class="mr-2">
                    <label for="mqtt" class="block text-sm font-medium text-white">Topic</label>
                  </div>

                  <div>
                    <div class="mt-1 mr-1 flex items-center space-x-2">
                      <input
                        id="mqtt"
                        v-model="localItem['mqttTopic']"
                        class="flex-grow px-3 py-2 w-60 bg-white border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-indigo-500 focus:border-indigo-500 sm:text-sm"
                      />
                      <SelectList
                        :topicList="topicList"
                        :initialSelected="localItem.mqttTopic"
                        @update:selected="updateMqttTopic"
                      />
                    </div>
                  </div>
                </div>
              </div>
            </div>
          </div>
          <!--footer-->
          <div
            class="mt-4 flex items-center justify-end p-6 border-t border-solid border-blueGray-200 rounded-b"
          >
            <button
              class="text-green-500 background-transparent font-bold uppercase px-6 py-2 text-sm outline-none focus:outline-none mr-1 mb-1 ease-linear transition-all duration-150"
              type="button"
              v-on:click="handleModal()"
            >
              Apply
            </button>
            <button
              class="text-red-500 background-transparent font-bold uppercase px-6 py-2 text-sm outline-none focus:outline-none mr-1 mb-1 ease-linear transition-all duration-150"
              type="button"
              v-on:click="toggleModal()"
            >
              Close
            </button>
          </div>
        </div>
      </div>
    </div>
    <div v-if="showModal" class="opacity-25 fixed inset-0 z-40 bg-black"></div>
  </div>
</template>

<script>
import { ref, watch, nextTick, reactive, onMounted } from 'vue'
import {
  Engine,
  Scene,
  HemisphericLight,
  ArcRotateCamera,
  Vector3,
  SceneLoader
} from '@babylonjs/core'
import { useRouter } from 'vue-router'
import axios from 'axios'
import SelectList from '../lists/SelectList.vue'
import emitter from '../eventBus'

export default {
  components: { SelectList },
  props: {
    item: {
      type: Object,
      default: () => ({})
    }
  },
  setup(props, { emit }) {
    const bjsCanvas = ref(null)
    let engine, scene, camera
    const router = useRouter()

    const settings = ref(['name', 'mqttTopic', 'threshold'])
    const topicList = ref([])
    const fetchTopicList = async () => {
      try {
        const response = await axios.get('http://traum.groundkim.com:3001/influx/sensor/topic/list')

        topicList.value = response.data
      } catch (error) {
        console.error('Failed to fetch blackbox list', error)
      }
    }

    const createScene = (canvas) => {
      engine = new Engine(canvas, true, { stencil: true })
      scene = new Scene(engine)
      // 초기 확대 수준을 조정 (5에서 3으로 변경하여 더 가까이 보이도록 설정)
      camera = new ArcRotateCamera('camera', -Math.PI / 2, Math.PI / 2.5, 3, Vector3.Zero(), scene)
      camera.attachControl(canvas, true)
      new HemisphericLight('light', new Vector3(0, 1, 0), scene)
      engine.runRenderLoop(() => {
        scene.render()
      })

      window.addEventListener('resize', () => {
        engine.resize()
      })

      return scene
    }

    const showModal = ref(false)
    const localItem = reactive({})

    const toggleModal = () => {
      showModal.value = !showModal.value
      if (showModal.value) {
        // 모달이 열릴 때 localItem을 props.item으로 초기화
        Object.assign(localItem, props.item)
      }
      if (!showModal.value && props.item.meshName === 'sensor') {
        emitter.emit('resetLocation', props.item)
      }
    }

    const putSensorData = async () => {
      try {
        if (props.item.meshName === 'sensor') {
          await axios.put(`http://traum.groundkim.com:3001/sensor/object/${localItem.meshId}`, {
            glb_file_name: localItem.meshName,
            location: localItem.location,
            threshold: localItem.threshold,
            mqttTopic: localItem.mqttTopic
          })
        }
      } catch (error) {
        console.error('Failed to put sensor data', error)
      }
    }

    const handleModal = () => {
      emit('setCondition', localItem)
      toggleModal()
      putSensorData()
    }

    const routeToUnity = () => {
      router.push('/digitaltwin/1')
    }

    const meshName = props.item.meshName

    watch(showModal, (newValue) => {
      if (newValue) {
        nextTick(() => {
          scene = createScene(bjsCanvas.value)
          scene.activeCamera.alpha += Math.PI

          SceneLoader.ImportMesh('', './models/', `${meshName}S.glb`, scene, function (meshes) {
            meshes.forEach((mesh) => {
              switch (meshName) {
                case 'eduKit':
                  mesh.scaling = new Vector3(0.3, 0.3, 0.3) // x, y, z 축으로 2배 확대
                  break
                case 'sensor':
                  mesh.scaling = new Vector3(1, 1, 1) // x, y, z 축으로 2배 확대
                  break
              }

              // 모델의 중심으로 카메라 타겟을 조정
              let boundingInfo = mesh.getBoundingInfo()
              let center = boundingInfo.boundingBox.centerWorld
              camera.target = center

              // 모델의 크기에 따라 카메라 반지름 조정
              let extend = boundingInfo.boundingBox.extendSizeWorld
              let radius = Math.max(extend.x, extend.y, extend.z) * 50
              camera.radius = radius
            })
          })
        })
      } else {
        if (engine) {
          engine.dispose()
        }
      }
    })

    const updateMqttTopic = (selectedTopic) => {
      localItem.mqttTopic = selectedTopic
    }
    onMounted(fetchTopicList)

    return {
      showModal,
      toggleModal,
      handleModal,
      routeToUnity,
      bjsCanvas,
      settings,
      localItem,
      updateMqttTopic,
      topicList
    }
  }
}
</script>

<style scoped>
.tree ul {
  list-style-type: none;
  padding-left: 20px;
}
</style>
