<template>
<div class="max-w-lg m-auto mt-10 bg-blue-500 p-6"> 
    <h1 class="text-center text-2xl font-bold">記憶配對遊戲</h1>
    <div class="flex justify-between my-5">
        <div class="border rounded border-2 border-white p-2 bg-blue-800 text-white font-bold text-xl"> 剩餘 {{ timeRemaining }}秒 </div>
        <button 
            id="restartButton" 
            @click="resetGame()"
            class="border rounded ring-2 ring-white p-2 hover:bg-blue-800 hover:text-white transition duration-500 font-bold text-xl">重新開始
        </button>   
    </div>
    <div class="flex justify-center">
        <div id="container" class="grid grid-cols-4 gap-4" v-if="showCards">
           <div 
            v-for="(cardData,index) in shuffledSymbols" 
            :key="index"
            @click="handleCardClick(cardData,index)"
            class="card-wrapper"
            :class="{
                flipped:cardData.isFlipped,
                matched:cardData.isMatched,
                
            }"

           >
                <!-- 背面:? -->
                <div class="card-face back-face">?</div>
                <!-- 正面:圖案 -->
                <div class="card-face front-face">{{cardData.symbol}}</div>
            
           </div>
        </div>
    </div>
    <p :class="{'text-red-500 text-lg font-bold text-center bg-white p-3 mt-6':message}">{{ message }}</p> 
</div> 
</template>

<script setup>
import { ref,onMounted,nextTick } from 'vue'

const symbols = ['🍎', '🍌', '🥝', '🍉', '🍇', '🍍', '🍓', '🍋']
const cardSymbols = symbols.flatMap( symbol => [ symbol , symbol ])//複製商品

const shuffledSymbols = ref([])//轉換格式後的隨機牌
const isGameLocked = ref(false) //鎖定所有卡片的點擊輸入，在玩家翻開第二張牌後、等待結果的期間，避免翻到第3張牌。
const firstCardIndex = ref(null)
const secondCardIndex = ref(null)

const timeRemaining = ref(60)//秒(遊戲時間)
const timerInterval = ref(null); //用來儲存 setInterval 的 ID,clearInterval()會用到。
const message = ref('')//訊息的顯示
const showCards = ref(true);
//洗牌
const shuffleArray = (array) => {
    for ( let i = array.length - 1 ; i > 0 ; i--) {
        const j = Math.floor(Math.random() * ( i + 1 ) )
        // 使用解構賦值電腦出現錯誤
        // [array[i],array[j]] = [array[j],array[i]]
        //使用傳統方式
        const temp = array[i]; 
        array[i] = array[j]; 
        array[j] = temp; 
    }
    return array
}

// 遊戲牌的初始狀態
const initializeGame = () => {
    //隨機牌 <note>使用[...cardSymbols]創建一個 全新的陣列副本，維持公平性 
    const tempShuffled = shuffleArray([...cardSymbols]) 
    //加入isFlipped、isMatched
    shuffledSymbols.value = tempShuffled.map((symbol,index) => ({
        index:index,
        symbol:symbol,
        isFlipped:false,//未掀牌
        isMatched:false,//未配對
    }))
    //卡片狀態重設
    resetTurn()
}


// 誰被點擊用index(設變數)、被點擊的資料內容、追蹤卡片翻牌沒翻牌狀態(設變數)
// 第1張牌和第2張牌配對
// 已配對的牌不能點擊 (設變數)、已經掀開的牌不能點擊 (設變數)  退出函式

const handleCardClick = (cardData,index) => {
    //檢查
    if(isGameLocked.value || cardData.isFlipped || cardData.isMatched) {
        return
    }
    //更新DOM和JavaScript數據
    cardData.isFlipped = true

    //點擊邏輯
    if( firstCardIndex.value === null ){
        //這是第一張牌
        firstCardIndex.value = index   
        startTimer()

    }else{
        //這是第二張牌
        secondCardIndex.value = index
        isGameLocked.value = true

        checkMatch()
    } 
}

//取得資料 / 比較資料 / 執行成功或失敗 / 清理狀態
const checkMatch = () => {
    //取得數據物件
    const card1 = shuffledSymbols.value[firstCardIndex.value]
    const card2 = shuffledSymbols.value[secondCardIndex.value]

    if (card1.symbol === card2.symbol) {
        setTimeout(() => {
            card1.isMatched = true
            card2.isMatched = true 

            //重置牌資料
            resetTurn()
            checkAllMatchedAndEnd()
        },1000)
        
    } else {
        setTimeout(() => {
            //數據更新
            card1.isFlipped = false
            card2.isFlipped = false

            //重置牌資料
            resetTurn()
        },1000)
    }
    
}

//重置牌資料
const resetTurn = () => {
    firstCardIndex.value = null
    secondCardIndex.value = null
    isGameLocked.value = false
}

//數據、視覺、終止條件(每秒發生的事情)
const updateTimer = () => {
    timeRemaining.value --;
    //
    if(timeRemaining.value <= 0 ) {
        isGameLocked.value = true
        clearInterval(timerInterval.value)
        message.value = '挑戰失敗'
    }
    
}

//啟動計時器
const startTimer = () => {
    //有ID就清除(清除舊的計時器 )
    if( timerInterval.value ) {
        return
    }
    // 創建一個計時器
    timerInterval.value = setInterval(updateTimer,1000)

}

//檢查所有配對並執行勝利函式
const checkAllMatchedAndEnd = () => {
    const allMatched = shuffledSymbols.value.every(card => card.isMatched ) 

    if(allMatched) {
        //執行勝利函式
        handleWin()
        return true
    }
    return false

}

//勝利後的操作
const handleWin = () => {
    //停止計時器
    stopTimer()
    message.value = '挑戰成功'
    isGameLocked.value = true
}

 //遊戲成功時間暫停
const stopTimer = () => {
    if(timerInterval.value) {
        //取消重複任務，時間會暫停
        clearInterval(timerInterval.value)
    }
}

//重新開始
const resetGame = () => {
    stopTimer()
    timerInterval.value = null // 確保計時器 ID 被清除，讓 startTimer 可以再次啟動
    timeRemaining.value = 60
    message.value = ''

    showCards.value = false;
    // initializeGame()  
    nextTick(() => {
        // 執行初始化，生成新的 shuffledSymbols 數據
        initializeGame();
        
        // 瞬間重建卡片 DOM
        showCards.value = true;
    });
} 

onMounted(() => {
    //顯示卡片
    initializeGame()
})
</script>
<style scoped>
.card-wrapper {
    width:100px;
    height:100px;
    position:relative;
    transform-style:preserve-3d;
    transition:transform 0.5s;
    cursor:pointer;
    border-radius:8px;
}

.card-wrapper.flipped {
    transform:rotateY(180deg);
}
.card-wrapper.matched {
    opacity:0.5;
    /* 配對成功後，卡片永遠不能再被點擊 */
    pointer-events:none;
}
.card-face {
    position:absolute;
    width: 100%;
    height:100%;
    backface-visibility:hidden;
    display:flex;
    align-items:center;
    justify-content:center;
    border-radius:8px;
    font-size:40px;
    font-weight:bold;
}  
/* 卡片背面顏色 */
.back-face {
    background-color:deepskyblue;
    color:aliceblue;
} 
/* 卡片正面顏色 */ 
.front-face {
    background-color:chartreuse;
    transform:rotateY(180deg);
}
</style>