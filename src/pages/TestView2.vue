<script setup lang="ts">
import { ref, computed, type Ref, watch, onBeforeUnmount, onMounted } from "vue";
import { useRoute, useRouter } from "vue-router";

import axios from "axios";

import { NIcon, NTag, NSpace, NSkeleton, NTooltip, NButton, NSwitch, useMessage, NA } from "naive-ui";
import { Checkmark, Send, ChevronBackSharp, SpeedometerOutline, Pencil, Shuffle, Cog, Newspaper, Close, Time, Menu, Navigate, Backspace, AppsSharp, ChatboxEllipses } from "@vicons/ionicons5";

const router = useRouter();
const route = useRoute();
const testId = route.params.id;

// 消息
const message = useMessage();

const loading = ref(true);

import type { ChooseProblemType, ProblemType } from "@/base/ProblemTypes";
import { hashCode } from "@/base/funcs";

type ProblemListType = {
    title: string;
    test: [number, number, number, number, number] | number;
    score: [number, number, number, number, number];
    problems: ProblemType[];

}
const problemInfo: Ref<ProblemListType> = ref({
    title: "加载中",
    test: [0, 0, 0, 0, 0],
    score: [0, 0, 0, 0, 0],
    problems: []
});

function backToHome() {
    router.push('/');
}

// 手机端查看的页面
// 0: 导航栏 1: 题目 2: 题目导航
const viewMode = ref(0);

const problemTypes = ref(["送分题", "单选题", "多选题", "填空题", "判断题"]);
const nowProblemList: Ref<ProblemType[]> = ref([]);
// 记录作答题目的和当前题目的答案
const answerList: Ref<(number | string)[][]> = ref([]);
const nowAnswer: Ref<(number | string)[]> = ref([]);
const writeAnswer = ref("");
// 记录所有题目的状态
const problemState: Ref<number[]> = ref([]);
// 错题
const errorProblems: Ref<ProblemType[]> = ref([]);
// 记录当前题目中的正确和错误答案
const rightAnswer: Ref<(number | string)[]> = ref([]);
const wrongAnswer: Ref<(number | string)[]> = ref([]);

const nowProblemId = ref(0);

// 反馈
function feedback() {
    message.info("请加QQ: 1440169768!");
}

// 用时
const timeCount: Ref<number> = ref(0);
const timeCounter: any = ref(null);

function timeFormat(e: number) {
    let res = ""
    let min = Math.floor(e / 60)
    let sec = e % 60
    if (min < 10)
        res = res + '0'
    res = res + String(min)
    res += "分"
    if (sec < 10)
        res = res + '0'
    res = res + String(sec)
    res += "秒"
    return res
}

// 得分
const testMode: Ref<number[] | number> = ref([0, 0, 0, 0, 0]); // 每种题型的题数
const score = ref(0);
const scoreList = ref([0, 0, 0, 0, 0]);
const pre = ref([0, 0, 0, 0, 0]);

// 0: Normal 1: Random 2: Custom 4: Error 5: Test 6 :Test Finished
const practiceModeName = ref(["顺序练习", "乱序练习", "自定义练习", "不知道", "错题回顾", "模拟考试", "模拟考试"])
const practiceMode = ref(0);
// PracticeMode和Mode合并为一个变量
const setPractice = ref(false); // 是否自定义题库
const setType: Ref<[boolean, boolean, boolean, boolean, boolean]> = ref([true, true, true, true, true]);
const setShuffle = ref(false);

const choices = ["A", "B", "C", "D", "E", "F", "G", "H", "I", "J", "K", "L", "M", "N"];

function changeProblemId(id: number) {
    nowProblemId.value = id;
    viewMode.value = 1;
}

// 正确数量
const correct = computed(() => {
    return problemState.value.filter(v => v == 2).length;
})
// 尝试次数
const attempted = computed(() => {
    return problemState.value.filter(v => v >= 2).length;
})
// 计算正确率
const rightRate = computed(() => {
    let attempted = 0;
    let correct = 0;

    problemState.value.forEach(v => {
        if (v >= 2) {
            attempted++;
            if (v === 2) {
                correct++;
            }
        }
    });

    if (attempted === 0) {
        return "0.00";
    }

    return (correct / attempted * 100).toFixed(2);
})

// 数据存储与提取
function initData() {
    answerList.value = [];
    problemState.value = [];
    answerList.value = nowProblemList.value.map(() => []);
    problemState.value = Array(nowProblemList.value.length).fill(0);
    nowProblemId.value = 0;
    setPractice.value = false;
    testMode.value = [0, 0, 0, 0, 0];
}
function loadData() {
    if (localStorage.vtixSaved == problemInfo.value.title) {
        //console.log("OK")
        nowProblemList.value = JSON.parse(localStorage.getItem("nowProblemList") || "");
        problemState.value = JSON.parse(localStorage.getItem("problemState") || "");
        answerList.value = JSON.parse(localStorage.getItem("answerList") || "");
        testMode.value = JSON.parse(localStorage.getItem("testMode") || "0") || 0;
        errorProblems.value = JSON.parse(localStorage.getItem("errorProblems") || "");
        practiceMode.value = Number(localStorage.getItem("practiceMode")) || 0;
        nowProblemId.value = Number(localStorage.getItem("nowProblemId")) || 0;

        if (nowProblemList.value[nowProblemId.value].type === 3) {
            if (answerList.value[nowProblemId.value].length > 0)
                writeAnswer.value = (answerList.value[nowProblemId.value][0] as string);
            else
                writeAnswer.value = "";
        }
        if (problemState.value[nowProblemId.value] > 1) {
            problemState.value[nowProblemId.value] = -1;
            submitAnswer();
        }
    }
    else {
        normalPractice();
    }
}
function storeData() {
    if (practiceMode.value === 5)
        return;
    localStorage.setItem("vtixSaved", problemInfo.value.title);
    localStorage.setItem("nowProblemList", JSON.stringify(nowProblemList.value));
    localStorage.setItem("nowProblemId", String(nowProblemId.value));
    localStorage.setItem("practiceMode", String(practiceMode.value));
    localStorage.setItem("testMode", JSON.stringify(testMode.value));
    localStorage.setItem("problemState", JSON.stringify(problemState.value));
    localStorage.setItem("answerList", JSON.stringify(answerList.value));
    localStorage.setItem("errorProblems", JSON.stringify(errorProblems.value));
}

// 做题
// 选中选项
function choose(k: number) {
    if (problemState.value[nowProblemId.value] > 1)
        return
    if (nowProblemList.value[nowProblemId.value].type === 1 || nowProblemList.value[nowProblemId.value].type === 4) {
        nowAnswer.value = [k]
        submitAnswer()
    }
    else {
        if (nowAnswer.value.indexOf(k) !== -1) {
            nowAnswer.value = nowAnswer.value.filter(v => v !== k)
        }
        else
            nowAnswer.value.push(k)
    }
}
// 提交指定题目的答案
function submitAnswer(d: number = -1) {
    let checkProblemId = d
    if (checkProblemId === -1)
        checkProblemId = nowProblemId.value
    if (practiceMode.value === 5) {
        if (nowProblemList.value[checkProblemId].type === 3) {
            answerList.value[checkProblemId] = [String(writeAnswer.value)]
            writeAnswer.value = ""
        }
        else
            answerList.value[checkProblemId] = nowAnswer.value
        problemState.value[checkProblemId] = 1
        nextProblem();
    }
    else {
        checkAnswer();
    }
}
function checkAnswer(d: number = -1) {
    rightAnswer.value = []
    wrongAnswer.value = []
    let checkProblemId = d
    if (d !== -1)
        nowAnswer.value = answerList.value[d];
    if (checkProblemId === -1)
        checkProblemId = nowProblemId.value
    if (problemState.value[checkProblemId] > 1)
        return
    let flag = true
    if (problemState.value[checkProblemId] === -1)
        flag = false
    let passed = 0

    const checkProblem = nowProblemList.value[checkProblemId];
    if (checkProblem.type === 1 || checkProblem.type === 4) {
        for (const ans of nowAnswer.value) {
            if (ans === checkProblem.answer) {
                if (passed !== 2)
                    passed = 1
                rightAnswer.value.push(ans)
            }
            else {
                wrongAnswer.value.push(ans)
                passed = 2
            }
        }
        if (passed !== 1)
            rightAnswer.value.push(checkProblem.answer)
    }
    else if (checkProblem.type === 2) {
        const correctAnswers = new Set(checkProblem.answer);
        const userAnswers = new Set(nowAnswer.value);

        correctAnswers.forEach(answer => {
            if (!userAnswers.has(answer)) {
                rightAnswer.value.push(answer);
                wrongAnswer.value.push(answer);
                passed = 2;
            }
        });

        if (passed === 0) passed = 1;

        userAnswers.forEach(answer => {
            if (correctAnswers.has((answer as number))) {
                rightAnswer.value.push(answer);
            } else {
                wrongAnswer.value.push(answer);
                passed = 2;
            }
        });
    }
    else if (checkProblem.type === 3) {
        let p: string[] = []
        if (d != -1) {
            if (nowAnswer.value.length > 0) {
                nowAnswer.value[0] = (nowAnswer.value[0] as string).replace(/，/g, ",")
                p = nowAnswer.value[0].split(",")
            }
        }
        else {
            writeAnswer.value = writeAnswer.value.replace(/，/g, ",")
            p = writeAnswer.value.split(",")
        }
        p.map(v => v.trim());
        passed = 1
        let q = checkProblem.answer.split(",")
        q.map(v => v.trim());
        if (p.length != q.length)
            passed = 2
        else {
            for (let i = 0; i < p.length; ++i) {
                let tmp = q[i].split(";")
                tmp.map(v => v.trim());
                if (tmp.indexOf(p[i]) == -1) {
                    passed = 2
                    break
                }
            }
            //if (this.nowAnswer[0] === this.nowProblems[checkProblemId].answer)
            //    passed = 1
        }
        nowAnswer.value[0] = p.join(",")
    }
    answerList.value[checkProblemId] = nowAnswer.value
    if (passed === 1) {
        if (flag) {
            score.value += scoreList.value[checkProblem.type]
        }
        problemState.value[checkProblemId] = 2
        if (d === -1)
            nextProblem(flag)
    }
    else {
        if (flag && practiceMode.value !== 4) {
            errorProblems.value.push(checkProblem);
        }
        problemState.value[checkProblemId] = 3
    }
}

function getChoiceClass(cs: number) {
    // 需要优化
    // 打算整个题目的选项一起计算而不是一个一个算。
    // 首先进入题目后，如果是选择题，就全部初始化为default
    // 然后选择选项或者判断答案后，更新整个题目的选项
    return {
        chosen: nowAnswer.value.indexOf(cs) !== -1,
        wrong: wrongAnswer.value.indexOf(cs) !== -1,
        right: rightAnswer.value.indexOf(cs) !== -1
    }
}
function getProblemStateType(id: number) {
    const typeArr = ['default', 'info', 'success', 'error'] as ('default' | 'info' | 'success' | 'error')[];
    if (id === nowProblemId.value && problemState.value[id] === 0)
        return 'info';
    else
        return typeArr[problemState.value[id]];
}

function nextProblem(flag: boolean = true) {
    if (nowProblemId.value + 1 < nowProblemList.value.length && flag) {
        setTimeout(() => {
            nowProblemId.value += 1
        }, 200)
    }
}

function onIdChange(n: number, o: number): void {
    // 考试模式上一道题是填空题，还得保存填空题答案
    if (practiceMode.value === 5 && nowProblemList.value[o].type === 3) {
        answerList.value[o] = [writeAnswer.value];
        if (writeAnswer.value.length > 0)
            problemState.value[o] = 1;
    }
    // 更新答案信息
    nowAnswer.value = answerList.value[n];
    rightAnswer.value = [];
    wrongAnswer.value = [];
    if (nowProblemList.value[n].type === 3) {
        if (nowAnswer.value.length > 0)
            writeAnswer.value = (nowAnswer.value[0] as string)
        else
            writeAnswer.value = ""
    }
    if (problemState.value[n] > 1) {
        problemState.value[n] = -1
        submitAnswer()
    }
}
watch(() => nowProblemId.value, onIdChange);

onMounted(() => {
    window.addEventListener("unload", storeData);
})

onBeforeUnmount(() => {
    storeData();
    window.removeEventListener("unload", storeData);
})

// 首先从后端获取题目信息
axios.get(`/data/${testId}.json`).then(res => {
    problemInfo.value = res.data;
    loading.value = false;
    try {
        loadData();
        viewMode.value = 1;
    }
    catch (e) {
        //console.log(e);
        initData();
        errorProblems.value = [];
        viewMode.value = 0;
    }
}).catch(() => {
    // 出现错误直接404
    router.push("/notfound");
})

// 作答模式
function normalPractice() {
    practiceMode.value = 0;
    nowProblemList.value = [...problemInfo.value.problems];
    initData();
    viewMode.value = 1;
}
function shuffleProblem(problemList: ProblemType[]) {
    const randomSort = () => Math.random() - 0.5;

    problemList.sort(randomSort);

    // 随机答案
    problemList.forEach(problem => {
        if (problem.type === 1) {
            const answerHash = hashCode(problem.choices[problem.answer]);
            problem.choices.sort(randomSort);
            problem.answer = problem.choices.findIndex(choice => hashCode(choice) === answerHash);
        } else if (problem.type === 2) {
            const answerHashes = problem.answer.map(index => hashCode(problem.choices[index]));
            problem.choices.sort(randomSort);
            problem.answer = problem.choices.reduce((indices: number[], choice, index) => {
                if (answerHashes.includes(hashCode(choice))) {
                    indices.push(index);
                }
                return indices;
            }, []);
        }
    });
}
function shufflePractice() {
    practiceMode.value = 1;
    nowProblemList.value = [...problemInfo.value.problems];
    shuffleProblem(nowProblemList.value);

    initData();
    viewMode.value = 1;
}
function testPractice() {
    // 清除上次作题痕迹
    initData();
    
    shufflePractice();
    practiceMode.value = 5;
    testMode.value = problemInfo.value.test;
    scoreList.value = problemInfo.value.score;

    if (typeof testMode.value === "number") {
        nowProblemList.value = nowProblemList.value.slice(0, testMode.value);
    } else {
        const nowProblemsCopy = [...nowProblemList.value];
        nowProblemList.value = [];

        testMode.value.forEach((count, type) => {
            const problemsOfType = nowProblemsCopy.filter(problem => problem.type === type);
            nowProblemList.value.push(...problemsOfType.slice(0, count));
        });

        pre.value = testMode.value.reduce((acc: number[], count, index) => {
            acc[index] = (acc[index - 1] || 0) + ((testMode.value as number[])[index - 1] || 0);
            return acc;
        }, []);
    }

    timeCount.value = 0;
    score.value = 0;

    clearInterval(timeCounter.value);
    timeCounter.value = setInterval(() => {
        timeCount.value += 1;
    }, 1000);
}
function errorPractice() {
    if (errorProblems.value.length == 0) {
        message.error("暂无错题!");
        return
    }
    practiceMode.value = 4;
    nowProblemList.value = [...errorProblems.value];
    initData();
    viewMode.value = 1;
}
function customPractice() {
    const nowProblemsCopy = [...problemInfo.value.problems];
    nowProblemList.value = nowProblemsCopy.filter(problem => setType.value[problem.type]);

    if (nowProblemList.value.length === 0) {
        message.error("这样子设定就没有题目可以做了呢");
        return;
    }

    setPractice.value = false;

    if (setShuffle.value) {
        shuffleProblem(nowProblemList.value);
    }

    practiceMode.value = 2;
    initData();
    viewMode.value = 1;
}
function startCustomPractice() {
    setPractice.value = true;
    viewMode.value = 1;
}
let lastManage = 0;
function deleteThisProblem(id: number) {
    if ((new Date().getTime()) - lastManage <= 120000 || confirm("真的要移除这道题吗？")) {
        lastManage = new Date().getTime()
        if (id < 0 || id >= errorProblems.value.length)
            return
        if (errorProblems.value.length == 1) {
            errorProblems.value = []
            message.info("错题已经全部清空!")
            normalPractice();
            return
        }
        problemState.value.splice(id, 1);
        answerList.value.splice(id, 1);
        errorProblems.value.splice(id, 1);
        nowProblemList.value.splice(id, 1);
        // 删掉的是最后一题，要往前一题
        if (id === errorProblems.value.length)
            --nowProblemId.value;
        else {
            onIdChange(nowProblemId.value, nowProblemId.value);
        }
    }
}
// 交卷
function submitPaper() {
    for (let i = nowProblemList.value.length - 1; i >= 0; --i)
        checkAnswer(i);
    practiceMode.value = 6;
    nowAnswer.value = answerList.value[nowProblemId.value];
    clearInterval(timeCounter.value);
}
</script>

<template>
    <div class="v-container">
        <div class="v-ti-subheader link" @click="backToHome"><n-icon :component="ChevronBackSharp"></n-icon> <span>vtix
                答题自测</span></div>
        <h2 class="v-ti-header">{{ problemInfo.title }}</h2>
    </div>
    <div class="v-divider"></div>
    <div class="v-nav hidden" :class="{ 'show': viewMode === 0 }">
        <div class="v-nav-container v-container">
            <div class="nav-item mobile-only">
                <div class="nav-link" @click="viewMode = 1">
                    <n-icon :component="Backspace" size="16"></n-icon>返回练习
                </div>
            </div>
            <div class="nav-item">
                <div class="nav-link primary" :class="{ 'active': practiceMode === 0 && !setPractice }"
                    @click="normalPractice">
                    <n-icon :component="Pencil" size="16"></n-icon>顺序练习
                </div>
            </div>
            <div class="nav-item">
                <div class="nav-link primary" :class="{ 'active': practiceMode === 1 && !setPractice }"
                    @click="shufflePractice"><n-icon :component="Shuffle" size="18"></n-icon>乱序练习
                </div>
            </div>
            <div class="nav-item">
                <div class="nav-link primary" :class="{ 'active': setPractice || practiceMode === 2 }"
                    @click="startCustomPractice"><n-icon :component="Cog" size="18"></n-icon>自定义练习 </div>
            </div>
            <div class="nav-item">
                <div class="nav-link red" @click="testPractice"
                    :class="{ 'active': practiceMode >= 5 && !setPractice }">
                    <n-icon :component="Newspaper" size="16"></n-icon>模拟测试
                </div>
            </div>
            <div class="nav-item">
                <div class="nav-link orange" :class="{ 'active': practiceMode === 4 && !setPractice }"
                    @click="errorPractice"><n-icon :component="Close" size="18"></n-icon>错题回顾 </div>
            </div>
            <div class="nav-item mobile-only">
                <div class="nav-link" @click="feedback"><n-icon>
                        <ChatboxEllipses />
                    </n-icon><span>意见反馈</span> </div>
            </div>
            <div class="nav-item mobile-only">
                Made by <n-a href="https://github.com/yemaster" target="_blank">yemaster</n-a>
            </div>
        </div>
    </div>
    <div class="corner-buttons mobile-only">
        <div class="corner-button">
            <n-button strong secondary circle :type="viewMode === 2 ? 'error' : 'primary'" size="large" class=""
                @click="viewMode === 2 ? (viewMode = 1) : (viewMode = 2)">
                <template #icon>
                    <n-icon>
                        <Close v-if="viewMode === 2" />
                        <Navigate v-else />
                    </n-icon>
                </template>
            </n-button>
        </div>
        <div class="corner-button">
            <n-button strong secondary circle :type="viewMode === 0 ? 'error' : 'info'" size="large" class=""
                @click="viewMode === 0 ? (viewMode = 1) : (viewMode = 0)">
                <template #icon>
                    <n-icon>
                        <Close v-if="viewMode === 0" />
                        <Menu v-else />
                    </n-icon>
                </template>
            </n-button>
        </div>
    </div>
    <div class="v-problem-main" v-show="viewMode !== 0">
        <div class="v-container">
            <div class="v-row">
                <div class="v-col-8 hidden" :class="{ 'show': viewMode === 1 }">
                    <template v-if="loading">
                        <n-skeleton text size="medium" />
                        <n-skeleton text :repeat="4" />
                    </template>
                    <template v-else>
                        <div class="v-problem-info">
                            <template v-if="setPractice === false">
                                <div class="v-problem-header">
                                    <n-tag type="primary">{{ problemTypes[nowProblemList[nowProblemId].type]
                                        }}</n-tag>
                                    <span class="v-problem-title">
                                        &nbsp;{{ nowProblemId + 1 }}/{{ nowProblemList.length }}.
                                        <span v-text="nowProblemList[nowProblemId].content"></span>
                                    </span>
                                </div>
                                <div class="v-problem-answer">
                                    <template v-if="nowProblemList[nowProblemId].type === 3">
                                        <div class="form-group">
                                            <input class="v-input" :class="{
                                                'input-success': problemState[nowProblemId] === 2,
                                                'input-error': problemState[nowProblemId] === 3,
                                            }" v-model="writeAnswer" placeholder="多个答案之间用逗号隔开！">
                                        </div>
                                        <div v-if="problemState[nowProblemId] === 3" class="form-group">
                                            <input class="v-input input-success readonly" readonly
                                                :value="nowProblemList[nowProblemId].answer">
                                        </div>
                                    </template>
                                    <template v-else>
                                        <span class="v-choose-box"
                                            v-for="(c, i) in (nowProblemList[nowProblemId] as ChooseProblemType).choices"
                                            v-bind:key="i" @click="choose(i)" :class="getChoiceClass(i)">
                                            {{ choices[i] }}. {{ c }}</span>
                                    </template>
                                </div>

                                <div class="v-button-area">
                                    <n-space>
                                        <n-button strong secondary type="info"
                                            v-if="(problemState[nowProblemId] <= 1) && (nowProblemList[nowProblemId].type !== 1) && (nowProblemList[nowProblemId].type !== 4)"
                                            @click="submitAnswer()">
                                            确定
                                        </n-button>
                                        <n-button strong secondary v-if="nowProblemId > 0" @click="nowProblemId -= 1">
                                            上一题
                                        </n-button>
                                        <n-button strong secondary v-if="nowProblemId + 1 < nowProblemList.length"
                                            @click="nowProblemId += 1">
                                            下一题
                                        </n-button>
                                        <n-button strong secondary type="error" v-if="practiceMode === 4"
                                            @click="deleteThisProblem(nowProblemId)">
                                            移除错题
                                        </n-button>
                                        <n-button strong secondary type="primary" v-if="practiceMode === 5"
                                            @click="submitPaper">
                                            交卷
                                        </n-button>
                                    </n-space>
                                </div>
                            </template>
                            <template v-else>
                                <b class="v-switch-header">练习题型</b>
                                <div class="v-switch-group" v-for="(problemType, i) in problemTypes" v-bind:key="i">
                                    <n-switch v-model:value="setType[i]" />
                                    <span :class="setType[i] ? 'info' : 'grey'" @click="setType[i] = !setType[i]">{{
                                        problemType }}</span>
                                </div>
                                <br>
                                <b class="v-switch-header">练习设置</b>
                                <div class="v-switch-group">
                                    <n-switch v-model:value="setShuffle" />
                                    <span :class="setShuffle ? 'info' : 'grey'"
                                        @click="setShuffle = !setShuffle">打乱题目</span>
                                </div>
                                <div class="v-button-area">
                                    <n-space>
                                        <n-button strong secondary type="info" @click="customPractice">
                                            完成
                                        </n-button>
                                    </n-space>
                                </div>
                            </template>
                        </div>
                    </template>
                </div>
                <div class="v-col-4 hidden" :class="{ 'show': viewMode === 2 }">
                    <template v-if="loading">
                        <n-skeleton text size="small" :repeat="5" />
                    </template>
                    <template v-else>
                        <div class="v-problem-navigator">
                            <div class="v-status-text">
                                <template v-if="practiceMode === 5">
                                    <n-tooltip trigger="hover">
                                        <template #trigger>
                                            <n-tag type="info">
                                                {{ timeFormat(timeCount) }}
                                                <template #icon>
                                                    <n-icon><Time /></n-icon>
                                                </template>
                                            </n-tag>
                                        </template>
                                        用时
                                    </n-tooltip>
                                </template>
                                <template v-else-if="practiceMode === 6">
                                    <n-tooltip trigger="hover">
                                        <template #trigger>
                                            <n-tag type="warning">
                                                {{ timeFormat(timeCount) }}
                                                <template #icon>
                                                    <n-icon><Time /></n-icon>
                                                </template>
                                            </n-tag>
                                        </template>
                                        用时
                                    </n-tooltip>
                                    <n-tooltip trigger="hover">
                                        <template #trigger>
                                            <n-tag type="primary">
                                                {{ score }}分
                                                <template #icon>
                                                    <n-icon>
                                                        <Checkmark />
                                                    </n-icon>
                                                </template>
                                            </n-tag>
                                        </template>
                                        得分
                                    </n-tooltip>
                                </template>
                                <div v-else>
                                    <n-tooltip trigger="hover">
                                        <template #trigger>
                                            <n-tag type="success">
                                                {{ correct }}
                                                <template #icon>
                                                    <n-icon>
                                                        <Checkmark />
                                                    </n-icon>
                                                </template>
                                            </n-tag>
                                        </template>
                                        正确数
                                    </n-tooltip>
                                    <n-tooltip trigger="hover">
                                        <template #trigger>
                                            <n-tag type="info">
                                                {{ attempted }}
                                                <template #icon>
                                                    <n-icon>
                                                        <Send />
                                                    </n-icon>
                                                </template>
                                            </n-tag>
                                        </template>
                                        作答数
                                    </n-tooltip>
                                    <n-tooltip trigger="hover">
                                        <template #trigger>
                                            <n-tag type="warning">
                                                {{ rightRate }}%
                                                <template #icon>
                                                    <n-icon>
                                                        <SpeedometerOutline />
                                                    </n-icon>
                                                </template>
                                            </n-tag>
                                        </template>
                                        正确率
                                    </n-tooltip>
                                </div>
                            </div>
                            <div class="v-problem-navigator-container">
                                <template v-if="practiceMode >= 5 && typeof testMode !== 'number'">
                                    <template v-for="(problemNum, k) in testMode" v-bind:key="k">
                                        <template v-if="problemNum > 0">
                                            <div class="v-problem-secTitle">{{ problemTypes[k] }}({{
                                                scoreList[k] *
                                                problemNum }}分)
                                            </div>
                                            <n-space>
                                                <n-tag :type="getProblemStateType(g + pre[k] - 1)" class="link"
                                                    v-for="g in problemNum" v-bind:key="g"
                                                    @click="changeProblemId(g + pre[k] - 1)">
                                                    {{ g + pre[k] }}
                                                </n-tag>
                                            </n-space>
                                        </template>
                                    </template>
                                </template>
                                <n-space v-else>
                                    <n-tag :type="getProblemStateType(k - 1)" class="link"
                                        v-for="k in nowProblemList.length" v-bind:key="k" @click="changeProblemId(k - 1)">
                                        {{ k }}
                                    </n-tag>
                                </n-space>
                            </div>
                        </div>
                    </template>
                </div>
            </div>
        </div>
    </div>
    <div class="v-status-area mobile-only">
        <n-space>
            <n-tag @click="viewMode = 0" type="info" v-if="viewMode !== 0">
                {{ practiceModeName[practiceMode] }}</n-tag>
            <n-tag @click="viewMode = 1" v-else>返回答题</n-tag>
            <template v-if="practiceMode === 5">
                <n-tooltip trigger="hover">
                    <template #trigger>
                        <n-tag type="info">
                            {{ timeFormat(timeCount) }}
                            <template #icon>
                                <n-icon><Time /></n-icon>
                            </template>
                        </n-tag>
                    </template>
                    用时
                </n-tooltip>
            </template>
            <template v-else-if="practiceMode === 6">
                <n-tooltip trigger="hover">
                    <template #trigger>
                        <n-tag type="primary">
                            {{ score }}分
                            <template #icon>
                                <n-icon>
                                    <Checkmark />
                                </n-icon>
                            </template>
                        </n-tag>
                    </template>
                    得分
                </n-tooltip>
            </template>
            <div v-else>
                <n-tooltip trigger="hover">
                    <template #trigger>
                        <n-tag type="primary">
                            {{ rightRate }}%
                            <template #icon>
                                <n-icon>
                                    <Checkmark />
                                </n-icon>
                            </template>
                        </n-tag>
                    </template>
                    正确率
                </n-tooltip>
            </div>
            <n-tag @click="viewMode = 2" type="info" v-if="viewMode !== 2">
                {{ nowProblemId + 1 }}/{{ nowProblemList.length }}
                <template #icon>
                    <n-icon>
                        <AppsSharp />
                    </n-icon>
                </template></n-tag>
            <n-tag @click="viewMode = 1" v-else>返回答题</n-tag>
        </n-space>
    </div>
</template>