<template>
  <div class="bg"></div>
  <div class="settings">
    <Icon name="replay" @click="restart" />
    <!-- <Icon name="setting" @click="handleSetting"/> -->
  </div>
  <div class="box">
    <div class="app">
      <div class="cards-container">
        <div class="cards-inner">
          <div
            v-for="(item, index) in cards"
            :class="`cardItem ${item.isCover? 'cardItem-coverd':''}`"
            :key="item.id"
            :style="{
              transform: `translateX(${
                item.status === 0 ? item.x : sortedQueue[item.id]
              }%) translateY(${item.status === 0 ? item.y : 1000}%)`,
              opacity: item.status < 2 ? 1 : 0,
            }"
            @click="chooseCard(index)"
          >
            <div
              class="cardItem-inner"
            >
              <img style="width: 100%" :src="`/src/assets/icons/${item.icon}.png`" alt="" />
            </div>
          </div>
        </div>
      </div>
      <div class="card-cache"></div>
      <div class="queue-container"></div>
      <!-- <div class="flex-container flex-between"> -->
      <div class="option-container" >
        <Button class="option-btn" color="black" @click="moveOut">移出</Button>
        <Button class="option-btn" color="black" @click="undo" >撤销</Button>
        <Button class="option-btn" color="black" @click="wash">洗牌</Button>
      </div>
      <Overlay :show="finished">
        <div class="modal">
          <div style="width:100%">
            <h1>{{ tipText }}</h1>
            <Button class="option-btn" color="black" @click="restart" >再来一次</Button>
          </div>
        </div>
      </Overlay>
    </div>
  </div>
</template>

<script setup>
import { ref, watchEffect } from "vue";
import { v4 as uuid } from 'uuid';
import { Button, Icon,Overlay } from 'vant';
import 'vant/lib/index.css';

const options = {
  iconLength: 15,
  level: 3,
  range: [0, 8],
  minRange: [4, 5],
  offsetPool:[0, 50, -50] // 偏移量取值范围
}
const initCards = (options) => {
  const {level, range, iconLength, offsetPool, minRange } = options
  const cards = [];
  const randomSet = (level) => {
    const cardNums = level * 3 * iconLength;
    // 每种图案需要是3的倍数，生成随机卡池
    let iconIdxPool = [];
    for (let i = 0; i < cardNums / 3; i++) {
      const curIconIdx = 1 + Math.floor(iconLength * Math.random())
      iconIdxPool = iconIdxPool.concat(new Array(3).fill(curIconIdx))
    }
    let rangeLen = 0,
        curRange = minRange; // 【动态增加range】
    for (let i = 0; i < cardNums; i++){
      rangeLen= curRange[1] - curRange[0]
      // 求偏移量
      const offsetX = offsetPool[Math.floor(offsetPool.length * Math.random())];
      const offsetY = offsetPool[Math.floor(offsetPool.length * Math.random())];
      // 偏移求列数
      const row = curRange[0] + Math.floor(rangeLen * Math.random());
      // 求偏移行数
      const column = curRange[0] + Math.floor(rangeLen * Math.random());
      // 从iconIdxPool中随机取出一个
      const iconIdx = iconIdxPool.splice(Math.floor(iconIdxPool.length * Math.random()),1)[0]
      // 生成元数据对象
      cards.push({
        isCover: false, // 默认都是没有被覆盖的
        status: 0, // 是否被选中的状态
        icon: parseInt(iconIdx), // 图标
        id: uuid(), // 生成随机id
        x: column * 100 + offsetX, //x 坐标
        y: row * 100 + offsetY, // y坐标
      });
      // 增加难度：越是堆叠在下面的牌，range范围越小，在生成cards数组过程中逐渐增大range
      // 按牌数比例来计算当前的范围长度应该为多少
      const newRangeLen = Math.ceil(i/cardNums * (range[1] - range[0]))
      // 如果范围长度需要增加，则计算分curRange[0]、curRange[1]分别应改变多少 => 因为每次循环都计算，所以范围增加的总值一定是1，也就是curRange[0]--或curRange[1]++
      if (newRangeLen > rangeLen){
        if((Math.random() > 0.5 && curRange[0] - 1 >= 0) || curRange[1] + 1 > range[1]){
          curRange[0] --
        } else {
          curRange[1] ++
        }
      }
    }
  };
  randomSet(level);
  return cards;
}
//延时函数
const waitTimeout = (timeout) => {
  return new Promise((resolve) => {
    setTimeout(() => {
      resolve();
    }, timeout);
  });
};

// const options =
const cards = ref(initCards(options)); // 元数据
const level = ref(1); // 等级
const queue = ref([]); // 下方选中数据
const sortedQueue = ref({}); // 排序后的x坐标
const finished = ref(false); // 是否完成
const tipText = ref(""); // 提示
const animating = ref(false); // 选择的元素正在加入队列区

// 检查是否被覆盖
const checkCover = (cards) => {
  const updatecards = cards.slice();
  for (let i = 0; i < updatecards.length; i++) {
    const cur = updatecards[i];
    cur.isCover = false;
    //status=0的牌在牌堆区，不在牌堆区的牌不需要比较
    if (cur.status !== 0) continue;
    // 拿到坐标
    const { x: x1, y: y1 } = cur;
    // 每个棋盘格即卡牌的宽高视为100，拿到对角坐标
    const x2 = x1 + 100,
      y2 = y1 + 100;
    // 判断其“上面”的卡片是否覆盖它
    for (let j = i + 1; j < updatecards.length; j++) {
      const compare = updatecards[j];
      if (compare.status !== 0) continue;
      const { x, y } = compare;
      if (!(y + 100 <= y1 || y >= y2 || x + 100 <= x1 || x >= x2)) {
        cur.isCover = true;
        // 已经判断出当前元素被遮挡，跳出循环
        break;
      }
    }
  }
  cards.value = updatecards;
};


// 移出道具
const moveOut = () => {
  // 如果选中队列中少于3条数据那么就退出
  if (queue.value.length < 3) return;
  const updateQueue = queue.value.slice();
  updateQueue.sort((a,b) => sortedQueue.value[a.id] - sortedQueue.value[b.id])
  const moveOutCards = updateQueue.splice(0, 3)
  for (let i = 0; i < moveOutCards.length; i++) {
    const find = cards.value.find((s) => s.id === moveOutCards[i].id);
   if (find) {
      // 移出到单独区域，无遮挡
      find.status = 0;
      find.x = 100 * (i + 2);
      find.y = 870;
   }
  }
  queue.value = updateQueue;
};

// 撤销道具
const undo = () => {
  // 放回原来位置
  if (!queue.value.length) return;
  const updateQueue = queue.value.slice();
  const cardItem = updateQueue.pop();
  if (!cardItem) return;
  queue.value = updateQueue;
  cardItem.status = 0
  checkCover(cards.value);
};

// 洗牌道具
const wash = () => {
  // 队列区的牌不洗
  let iconPool = [];
  const queueIconMap = {}
  for (let queueItem of queue.value) {
    if (!queueIconMap[queueItem.icon]) {
      queueIconMap[queueItem.icon] = 1
    } else {
      queueIconMap[queueItem.icon]++
    }
  }
   // 匹配队列区的牌面，保证可以消除
  for (let icon in queueIconMap) {
    iconPool = iconPool.concat(new Array(3 - queueIconMap[icon]).fill(parseInt(icon)))
  }
  // 剩余牌（去除队列区和已匹配队列区的数量）重新赋予牌面，保证每种牌面数量都为3的倍数
  const restCardNum = cards.value.filter(item => item.status === 0).length;
  const fillNum = (restCardNum - iconPool.length) / 3;
  for (let i = 0; i < fillNum; i++) {
    const curIconIdx = 1 + Math.floor(options.iconLength * Math.random())
    iconPool = iconPool.concat(new Array(3).fill(curIconIdx))
  }
  const updateCards = []
  for (let item of cards.value) {
    if (item.status === 0) {
      updateCards.push({
        ...item,
        icon: iconPool.splice(Math.floor(iconPool.length * Math.random()), 1)[0]
      })
     } else {
      updateCards.push(item)
    }
  }
  cards.value = updateCards
};


// 重开
const restart = () => {
  finished.value = false;
  queue.value = [];
  cards.value = initCards(options)
  checkCover(cards.value);
};

// 选择卡片
const chooseCard = async (idx) => {
  // 游戏已完成/正在移动，直接return
  if (finished.value || animating.value) return;
  const cardItem = cards.value[idx];
  // 被覆盖/不在牌堆区，直接return
  if (cardItem.isCover || cardItem.status !== 0) return;
  //置为可以选中状态
  cardItem.status = 1;
  queue.value.push(cardItem);
  // 防止点击
  animating.value = true;
  //延迟判断三连匹配
  await waitTimeout(300);
  // 拿到与选中牌匹配的所有icon
  const filterSame = queue.value.filter((c) => c.icon === cardItem.icon);
  // 队列中该卡面已有三个，走消除逻辑
  if (filterSame.length === 3) {
    // 由于icon的类型一样，留下队列中的不一样的剩余内容重新赋值
    queue.value = queue.value.filter((c) => c.icon !== cardItem.icon);
    // 隐藏iocn，dom
    for (const sb of filterSame) {
      const find = cards.value.find((i) => i.id === sb.id);
      // 将他们的状态变为2 通过opacity 属性 来隐藏icon
      if (find) find.status = 2;
    }
  }

  // 当格子占满了，那么失败
  if (queue.value.length === 7) {
    tipText.value = "失败了";
    finished.value = true;
    animating.value = false;
    return;
  }
  // 所有cards的status置为2，完成挑战
  if (!cards.value.find((c) => c.status !== 2)) {
    tipText.value = '完成挑战';
    finished.value = true;
    animating.value = false;
    return;
  }
  // 处理覆盖情况
  checkCover(cards.value);
  // 正在移动中标志位置为false
  animating.value = false;
};

// 队列区排序
watchEffect(() => {
  const iconMap = {};
  // 通过当前的icon的标识，将相同的icon归纳到一块
  const temp = [];
  for (const cardItem of queue.value) {
    const icon = parseInt(cardItem.icon)
    if (typeof iconMap[icon] === 'number') {
      iconMap[icon] ++
      temp.splice(iconMap[icon], 0, cardItem)
    } else {
      temp.push(cardItem)
      iconMap[icon] = temp.length - 1;
    }
  }
  const updateSortedQueue = {};
  let x = 50;
  // 拿到更新后的队列区数据，计算坐标
  for (const cardItem of temp) {
    // 坐标
    updateSortedQueue[cardItem.id] = x;
    x += 100;
  }
  // 记录队列区牌顺序和位置
  sortedQueue.value = updateSortedQueue;
  // 检查覆盖情况
  checkCover(cards.value);
});
</script>
<style>
#app {
  text-align: center;
  width: 100vw;
  height: 100vh;
  position: relative;
  max-width: 500px;
}

.bg {
  position: absolute;
  background-image: url('@/assets/bg.png');
  background-size: 100% 100%;
  background-repeat: no-repeat;

  left: 0;
  right: 0;
  bottom: 0;
  top: 0;
}

.box {
  position: absolute;
  top: 0;
  left: 0;
  bottom: 0;
  right: 0;
  padding: 50px;
}

.settings{
  height: 2em;
  line-height: 2em;
  font-size: 1.5em;
  z-index: 2;
  z-index: 2;
  position: absolute;
  right: 0.5em;
}

.settings > i:hover{
  cursor: pointer;
  color:#999;
}

.settings > i:not(:last-child){
  margin-right: 0.2em;
  
}

.app {
  width: 100%;
  margin: 0 auto;
}

.cards-container {
  width: 100%;
  padding-bottom: 100%;
  position: relative;
  margin:12.5% 0 25% 0;
}

.cards-inner {
  position: absolute;
  left: 0;
  right: 0;
  bottom: 0;
  top: 0;
  overflow: visible;
}

.cardItem {
  width: 12.5%;
  padding-bottom: 12.5%;
  position: absolute;
  transition: 0.3s;
  left: 0;
  top: 0;
}
.cardItem-coverd > .cardItem-inner{
  background-color: #999;
}

.cardItem-inner {
  position: absolute;
  background-color: white;
  left: 0;
  right: 0;
  bottom: 0;
  top: 0;
  display: flex;
  justify-content: center;
  align-items: center;
  border-radius: 8px;
  border: 2px solid #444;
  transition: 0.3s;
  padding: 5px;
}

.queue-container {
  border-radius: 8px;
  width: 100%;
  padding-bottom: 15%;
  border: 2px solid gray;
  margin-top:-5px;
  margin-bottom: 16px;
  box-sizing: border-box;
  display: flex;
  gap: 8px;
  justify-content: center;
  align-items: center;
}



.modal {
  height: 100%;
  width:100%;
  display: flex;
  align-items: center;
  color:white;
  
}

.restart-btn{
  color:white;
}

.option-container{
  display:flex; 
  justify-content: space-between;
  align-items: center;
}
.option-btn{
  width:30%;
}
</style>