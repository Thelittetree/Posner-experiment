# Posner 实验简介

Posner 实验是一种常用于研究注意力的心理学实验，旨在探讨注意力如何影响反应时间。该实验通常包括一个固定点、一个提示和一个目标刺激。参与者的任务是根据目标的出现位置（左侧或右侧）进行反应。实验通过不同类型的提示（如方向性提示、闪光提示等）来考察其对目标识别的影响。

# 基于jsPsych的Posner 实验教程

## 明确实验设计思路

在实验中，我们需要的刺激材料有：固定的中心注视刺激，线索提示，目标刺激。其中，目标刺激为被试所需要判断位置并进行按键反应的刺激，在本实验中为蓝色方块。![目标刺激](images/target.png "目标刺激")

而线索为蓝色边框。![提示线索](images/blue%20square.png) 该边框可能出现在中心注视点左侧，也可能出现在中心注视点右侧。当蓝色边框出现在中心注视点左侧时，说明目标刺激更可能出现在中心注视点左侧；当蓝色边框出现在中心注视点右侧时，说明目标刺激更可能出现在中心注视点右侧。被试需要尽可能克服蓝色边框所带来的影响，又快又准地对目标刺激出现的位置进行按键反应。

除此之外，还有一个中性线索![中性线索](images/neutral%20cue.png)。当它出现时，说明目标刺激出现在左侧的概率同出现在右侧的概率相同。

## 1. 准备工作

### 1.1 引入必要的库

首先，我们需要引入 `jsPsych` 库及相关插件，同时引入用于生成 Excel 文件的库 `xlsx`。在 HTML 文件的 `<head>` 部分添加以下内容：

```html
<!DOCTYPE html>
<html>
<head>
    <title>Posner Experiment</title>
    <!-- 引入 jsPsych 库 -->
    <script src="https://unpkg.com/jspsych@7.3.4"></script>
    <script src="https://unpkg.com/@jspsych/plugin-html-keyboard-response@1.1.3"></script>
    <script src="https://unpkg.com/@jspsych/plugin-image-keyboard-response@1.1.3"></script>
    <script src="https://unpkg.com/@jspsych/plugin-preload@1.1.3"></script>
    <!-- 引入用于生成 Excel 文件的库 -->
    <script src="https://cdnjs.cloudflare.com/ajax/libs/xlsx/0.18.5/xlsx.full.min.js"></script>
    <link href="https://unpkg.com/jspsych@7.3.4/css/jspsych.css" rel="stylesheet" type="text/css" />
</head>
<body>
    <!-- 实验目标容器 -->
    <div id="jspsych-target"></div>
</body>
</html>
```

## 2. 创建实验

### 2.1 实验目标容器样式

使用 CSS 为实验页面创建基本样式，使其居中显示并适应不同的屏幕尺寸。将以下 CSS 添加到 `<style>` 标签中：

```html
<style>
    /* 基础 Flexbox 布局样式 */
    body {
        display: flex;
        justify-content: center;
        align-items: center;
        height: 100vh;
        margin: 0;
        font-family: Arial, sans-serif;
        background-color: #f0f0f0;
    }
    /* jsPsych 目标容器的样式 */
    #jspsych-target {
        display: flex;
        flex-direction: column;
        align-items: center;
        justify-content: center;
        width: 100%;
        max-width: 800px;
        height: 100%;
        position: relative;
        padding: 20px;
        box-sizing: border-box;
        background: #fff;
        border-radius: 8px;
        box-shadow: 0 0 10px rgba(0,0,0,0.1);
    }
    /* 方块和提示框的样式 */
    .square, .cue-box {
        width: 60px;
        height: 60px;
        position: absolute;
        top: calc(50% - 30px);
    }
    .square {
        background-color: blue;
    }
    .cue-box {
        border: 5px solid blue;
    }
    .fixation {
        position: absolute;
        top: 50%;
        left: 50%;
        transform: translate(-50%, -50%);
        font-size: 60px;
        color: black;
    }
    .left {
        left: calc(50% - 200px);
    }
    .right {
        left: calc(50% + 140px);
    }
    @media (max-width: 600px) {
        .square, .cue-box {
            width: 40px;
            height: 40px;
        }
        .left, .right {
            left: 50%;
            transform: translateX(-50%);
        }
    }
    /* 参与者信息表单样式 */
    #participant-info-form {
        display: flex;
        flex-direction: column;
        gap: 10px;
        align-items: flex-start;
    }
    #participant-info-form label {
        margin-bottom: 5px;
    }
    #submit-info {
        margin-top: 10px;
        padding: 5px 10px;
        background-color: #007BFF;
        color: white;
        border: none;
        border-radius: 5px;
        cursor: pointer;
    }
    #submit-info:hover {
        background-color: #0056b3;
    }
</style>
```

### 2.2 初始化 jsPsych 和实验时间线

在 `<script>` 标签中，我们初始化 `jsPsych` 并定义实验时间线。包括参与者信息收集、实验任务和数据保存：

```html
<script>
    /* 将数据保存为 Excel 文件的函数 */
    function saveDataAsExcel(filename, data) {
        var ws = XLSX.utils.json_to_sheet(data);
        var wb = XLSX.utils.book_new();
        XLSX.utils.book_append_sheet(wb, ws, 'Experiment Data');
        XLSX.writeFile(wb, filename);
    }

    /* 初始化 jsPsych */
    var jsPsych = initJsPsych({
        on_finish: function() {
            var data = jsPsych.data.get().values();
            var formattedData = data.map(function(d) {
                return {
                    trial: d.trial_index,
                    task: d.task,
                    response: d.response,
                    correct: d.correct,
                    rt: d.rt,
                    accuracy: d.correct ? 1 : 0
                };
            });
            saveDataAsExcel('experiment_data.xlsx', formattedData);
        },
        display_element: 'jspsych-target'
    });

    /* 创建时间线 */
    var timeline = [];

    /* 预加载图像 */
    var preload = {
        type: jsPsychPreload,
        images: []
    };
    timeline.push(preload);

    /* 定义参与者信息表单 */
    var participant_info = {
        type: jsPsychHtmlKeyboardResponse,
        stimulus: `
            <p>请输入以下信息：</p>
            <form id="participant-info-form">
                <label for="id">参与者 ID: </label><input name="id" type="text" required><br>
                <label for="age">年龄: </label><input name="age" type="number" required><br>
                <label for="hand">利手（"1" : 左利手, "2" : 右利手）: </label><input name="hand" type="text" required><br>
                <button type="button" id="submit-info">提交</button>
            </form>
            <p>按【提交】继续实验。</p>
        `,
        choices: "NO_KEYS",
        on_load: function() {
            document.getElementById('submit-info').addEventListener('click', function() {
                var form = document.getElementById('participant-info-form');
                var formData = new FormData(form);
                var responses = {};
                formData.forEach(function(value, key) {
                    responses[key] = value;
                });
                jsPsych.data.addProperties(responses);

                var experiment_timeline = [
                    /* 欢迎消息试验 */
                    {
                        type: jsPsychHtmlKeyboardResponse,
                        stimulus: "欢迎参加 Posner 实验。按任意键开始。"
                    },
                    /* 说明试验 */
                    {
                        type: jsPsychHtmlKeyboardResponse,
                        stimulus: `
                            <p>在本实验中，您需要将注意力保持在黑色的固定十字（+）上。</p>
                            <p>然后，会出现一个蓝色方块或黑色闪光图形，指示目标可能出现的位置。</p>
                            <p>如果蓝色方块轮廓出现在左侧，目标更可能出现在左侧，如果出现在右侧，目标更可能出现在右侧。</p>
                            <p>如果出现黑色闪光，目标在任一侧出现的可能性相等。</p>
                            <p><strong style="background-color: #ffff00; padding: 0 5px;">目标是一个蓝色方块。</strong></p>
                            <p>您的任务是：</p>
                            <p>在目标（蓝色方块）出现在左侧时，按【F】键。</p>
                            <p>在目标（蓝色方块）出现在右侧时，按【J】键，请尽快并尽可能准确地作答。</p>
                            <p>如果明白上述任务要求，请按任意键开始实验。</p>
                        `,
                        post_trial_gap: 2000
                    },
                    /* 固定点试验 */
                    {
                        type: jsPsychHtmlKeyboardResponse,
                        stimulus: '<div class="fixation">+</div>',
                        choices: "NO_KEYS",
                        trial_duration: 1000,
                        data: { task: 'fixation' }
                    },
                    /* 提示试验 */
                    {
                        type: jsPsychHtmlKeyboardResponse,
                        stimulus: function() {
                            var cue_type = jsPsych.timelineVariable('cue_type');
                            var fixation_html = cue_type === 'neutral_flash' ? '' : '<div class="fixation">+</div>';
                            return fixation_html + 
                                (cue_type === 'left_box' ? '<div class="cue-box left"></div>' : 
                                cue_type

 === 'right_box' ? '<div class="cue-box right"></div>' :
                                cue_type === 'neutral_flash' ? '<div style="background: black; width: 60px; height: 60px; position: absolute; top: calc(50% - 30px); left: calc(50% - 30px);"></div>' :
                                cue_type === 'neutral_arrow' ? '<div style="font-size: 60px; color: blue;">→</div>' : '');
                        },
                        choices: "NO_KEYS",
                        trial_duration: 1000,
                        data: { task: 'cue' }
                    },
                    /* 目标试验 */
                    {
                        type: jsPsychImageKeyboardResponse,
                        stimulus: '<div class="square"></div>',
                        choices: ['f', 'j'],
                        data: { task: 'target' },
                        on_finish: function(data) {
                            data.correct = (data.response === 'f' && data.cue_type === 'left_box') ||
                                (data.response === 'j' && data.cue_type === 'right_box');
                        }
                    }
                ];

                jsPsych.init({
                    timeline: experiment_timeline,
                    display_element: 'jspsych-target'
                });
            });
        }
    };
    timeline.push(participant_info);

    /* 添加时间线中的实验任务 */
    var trial = {
        timeline: [
            {
                type: jsPsychHtmlKeyboardResponse,
                stimulus: '<div class="fixation">+</div>',
                choices: "NO_KEYS",
                trial_duration: 1000,
                data: { task: 'fixation' }
            },
            {
                type: jsPsychHtmlKeyboardResponse,
                stimulus: function() {
                    var cue_type = jsPsych.timelineVariable('cue_type');
                    return (cue_type === 'left_box' ? '<div class="cue-box left"></div>' :
                        cue_type === 'right_box' ? '<div class="cue-box right"></div>' :
                        cue_type === 'neutral_flash' ? '<div style="background: black; width: 60px; height: 60px; position: absolute; top: calc(50% - 30px); left: calc(50% - 30px);"></div>' :
                        '<div style="font-size: 60px; color: blue;">→</div>');
                },
                choices: "NO_KEYS",
                trial_duration: 1000,
                data: { task: 'cue' }
            },
            {
                type: jsPsychImageKeyboardResponse,
                stimulus: '<div class="square"></div>',
                choices: ['f', 'j'],
                data: { task: 'target' },
                on_finish: function(data) {
                    data.correct = (data.response === 'f' && data.cue_type === 'left_box') ||
                        (data.response === 'j' && data.cue_type === 'right_box');
                }
            }
        ],
        timeline_variables: [
            { cue_type: 'left_box' },
            { cue_type: 'right_box' },
            { cue_type: 'neutral_flash' },
            { cue_type: 'neutral_arrow' }
        ],
        randomize_order: true
    };
    timeline.push(trial);

    /* 启动实验 */
    jsPsych.init({
        timeline: timeline,
        display_element: 'jspsych-target'
    });
</script>
```

## 3. 运行实验

将上述代码保存为一个 `.html` 文件，然后在浏览器中打开即可开始实验。确保在运行之前已经安装了必要的库，并且 HTML 文件中引入了这些库的正确 URL。

## 4. 结果查看

实验结束后，生成的 Excel 文件 `experiment_data.xlsx` 将自动下载到你的计算机。打开这个文件，你可以查看每次实验的详细数据，包括任务、响应、正确性、反应时间等信息。

## 总结

本教程介绍了如何使用 `jsPsych` 创建一个基本的 Posner 实验，包括参与者信息收集、实验任务设计、数据保存等步骤。通过这个教程，你能够大致明白如何通过jsPsych设计一个实验，并且掌握心理学实验的基本知识。

---

**附加资源：**

- [jsPsych 官方文档](https://www.jspsych.org/)
- [XLSX.js 文档](https://github.com/SheetJS/sheetjs)
- [Posner 实验简介](https://en.wikipedia.org/wiki/Posner_cueing_task)
- [Flexbox 布局教程](https://css-tricks.com/snippets/css/a-guide-to-flexbox/)
```