<template lang="pug">
  div(
    ref="container"
  )
</template>

<script>
import * as THREE from 'three'
import * as CANNON from 'cannon-es'

export default {
  data() {
    return {
      container: null,
    }
  },
  mounted() {
    const scene = new THREE.Scene()
    const camera = new THREE.PerspectiveCamera()
    const renderer = new THREE.WebGLRenderer({
      antialias: true,
    })

    // CANNON.jsの初期化
    const world = new CANNON.World()
    world.gravity.set(0, -9.82, 0) // 重力を設定

    //物体のリスト
    const objectList = []

    // 地面を作成
    const ground = {
      name: 'ground',
      mass: 0, // 地面は動かないので質量は0
      position: new CANNON.Vec3(0, 0, 0),
      shape: new CANNON.Box(new CANNON.Vec3(10, 0.01, 10)), // 地面の形状
      material: new CANNON.Material({
        restitution: 0.5, // 反発係数
      }),
      mesh: new THREE.Mesh(
        new THREE.BoxGeometry(20, 0.02, 20),
        new THREE.MeshStandardMaterial({
          color: 0x00ff00,
          transparent: true,
          opacity: 0.5,
        }),
      ),
    }
    objectList.push(ground)

    // 球体を作成
    const Sphere = {
      name: 'sphere',
      mass: 10, // 質量を設定
      position: new CANNON.Vec3(0, 10, 0), // 初期位置を設定
      shape: new CANNON.Sphere(0.5), // 球体の半径を0.5に設定
      material: new CANNON.Material({
        restitution: 1, // 反発係数
      }),
      mesh: new THREE.Mesh(
        new THREE.SphereGeometry(0.5, 32, 32),
        new THREE.MeshStandardMaterial({
          color: 0xff9999,
          transparent: true,
          opacity: 0.75,
        }),
      ),
    }
    objectList.push(Sphere)

    //objectListから生成した、bodyとmeshのリスト
    const bodyMeshList = []

    let i = 0
    for (const object of objectList) {
      // CANNON.jsの物理ボディを作成
      const body = new CANNON.Body({
        mass: object.mass,
        position: object.position,
        shape: object.shape,
        material: object.material,
      })

      // THREE.jsのメッシュを作成
      const mesh = object.mesh
      mesh.position.copy(body.position)
      mesh.quaternion.copy(body.quaternion)

      bodyMeshList.push({
        body: body,
        mesh: mesh,
      })
      world.addBody(bodyMeshList[i].body)
      scene.add(bodyMeshList[i].mesh)

      i++
    }

    const copy = () => {
      // CANNON.jsの物理ボディの位置と回転をTHREE.jsのメッシュにコピー
      for (const bodyMesh of bodyMeshList) {
        bodyMesh.mesh.position.copy(bodyMesh.body.position)
        bodyMesh.mesh.quaternion.copy(bodyMesh.body.quaternion)
      }
    }

    this.container = this.$refs.container

    // 環境光源を追加
    const directionalLight = new THREE.DirectionalLight(0xffffff, 1)
    directionalLight.position.set(0, 1, 0)
    scene.add(directionalLight)

    this.initThree(scene, camera, renderer, copy, world)
  },
  methods: {
    //世界を生成
    initThree(scene, camera, renderer, copy, world) {
      //DOMサイズ取得
      const clientWidth =
        window.innerWidth ||
        document.documentElement.clientWidth ||
        document.body.clientWidth
      const clientHeight =
        window.innerHeight ||
        document.documentElement.clientHeight ||
        document.body.clientHeight

      //グリッド追加
      scene.add(new THREE.GridHelper())

      //カメラ設定
      camera.aspect = clientWidth / clientHeight
      camera.updateProjectionMatrix()
      camera.position.set(30, 10, 0)
      camera.lookAt(0, 3, 0)

      //レンダラー設定
      renderer.setSize(clientWidth, clientHeight)
      renderer.setPixelRatio(clientWidth / clientHeight)
      this.$refs.container.appendChild(renderer.domElement)

      const animate = () => {
        const frame = () => {
          copy()
          world.fixedStep()
          renderer.render(scene, camera)
          requestAnimationFrame(frame)
        }
        frame()
      }

      animate()
    },
  },
}
</script>
