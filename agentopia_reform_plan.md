# Agentopia 历史唯物主义与系统优化重构方案 (Agentopia Reform & Optimization Plan)

本方案基于历史唯物主义对 Agentopia 系统的审计与京海中学（`school_07161448`）第一周真实运行数据的实证验证结果制定。旨在修复系统中存在的严重技术 Bug（如职位技能增长虚无、技能名中英文分裂），解构上层建筑的“绝对意识形态国家机器”统治，重构底层更具唯物主义因果特征的经济基础。

后续执行 Agent 可直接对照本方案中的具体步骤和代码行数进行重构开发。

---

## 目录
1. [修补生产力：修复每周职位技能增长结算 Bug](#1-修补生产力修复每周职位技能增长结算-bug)
2. [消除语言异化：解决中英文技能键名分裂问题](#2-消除语言异化解决中英文技能键名分裂问题)
3. [归还私有灵魂：解绑 judge_others 阶段的 God Model 强制格式化](#3-归还私有灵魂解绑-judge_others-阶段的-god-model-强制格式化)
4. [解耦唯心主义：God Model 语义因果与匹配算法重构展望](#4-解耦唯心主义god-model-语义因果与匹配算法重构展望)

---

## 1. 修补生产力：修复每周职位技能增长结算 Bug

### 【诊断结论】
在 `positions.json` 中给角色规划的技能增长数值（`weekly_delta_skills`）在每周结算时完全被遗漏。日常劳动无法沉淀为 Agent 自身的技能工具，这是严重的系统性生产力结算 Bug。

### 【重构方案】
在 `src/world/world.py` 的每周开始前阶段（`_before_week_start`），增加对每个活跃 Agent 职位技能增量的物理累加。

#### 1.1 修改位置
`src/world/world.py` 约第 447 行：

**修改前：**
```python
    def _before_week_start(self) -> None:
        """Execute all operations that should happen before each week starts."""
        self._apply_fulfillment_decay()
        self._settle_weekly_income()
```

**修改后：**
```python
    def _settle_weekly_skills(self) -> None:
        """Settle and accumulate weekly_delta_skills for all agents from their positions."""
        from src.utils import get_verify_logger
        verify_logger = get_verify_logger(feature="skills")
        
        if verify_logger:
            verify_logger.info("[VERIFY-SKILLS] Distributing weekly_delta_skills from positions")

        for agent in self.agents:
            profile = agent.dm.read_profile()
            position = profile.get("position", {})
            delta_skills = position.get("weekly_delta_skills", {})
            
            if delta_skills:
                # 获取当前最新状态（exclude_cur_t=False，防止时间混淆）
                state = agent.dm.read_state(exclude_cur_t=False)
                skills = state.setdefault("skills", {})
                
                # 记录旧数值供 verify_log 追溯
                old_skills = dict(skills)
                
                # 物理累加技能
                for skill_name, delta_val in delta_skills.items():
                    # 在累加前进行中文化/标准化校准（见重构方案第二条）
                    standard_key = self._standardize_skill_name(skill_name, agent.dm.language)
                    skills[standard_key] = skills.get(standard_key, 0) + delta_val
                
                # 存回状态账本
                agent.dm.save_state(state)
                
                if verify_logger:
                    changes = ", ".join(f"{k}: {old_skills.get(k, 0)} → {skills[k]} (+{delta_skills[k]})" for k in delta_skills)
                    verify_logger.info(f"[VERIFY-SKILLS] {agent.name}: {changes}")
                    
    def _standardize_skill_name(self, name: str, language: str) -> str:
        """Helper to map English LLM keys to Chinese if language is zh (and vice versa)."""
        # 实证中发现中英文分裂，需要进行映射处理
        from src.utils import standardize_skill_key
        return standardize_skill_key(name, language)

    def _before_week_start(self) -> None:
        """Execute all operations that should happen before each week starts."""
        self._apply_fulfillment_decay()
        self._settle_weekly_income()
        self._settle_weekly_skills()  # 注入技能物理结算
```

---

## 2. 消除语言异化：解决中英文技能键名分裂问题

### 【诊断结论】
底层世界模型在评估交互活动（Joint/Encounter）时返回英文的技能 key（例如 `"observation_and_empathy"`），而系统的本地角色卡采用中文（例如 `"观察与共情"`）。直接硬合并导致底层技能树词汇分裂，阻碍了后续技能对齐。

### 【重构方案】
在 `src/utils.py` 中实现一个双向映射和清洗机制 `standardize_skill_key()`，并在 DataManager 及 World 结算状态时，强制执行技能名中英文融合。

#### 2.1 修改位置 1：`src/utils.py` 末尾增加映射转换逻辑

```python
# 技能中英文映射字典
SKILL_TRANSLATION_MAP = {
    "zh": {
        "observation_and_empathy": "观察与共情",
        "observation": "观察力",
        "creativity": "创造力",
        "guitar_playing": "吉他演奏",
        "lyrics_creation": "歌词创作",
        "interpersonal_expression": "人际沟通",
        "cooking": "烹饪",
        "cooking_basics": "烹饪基础",
        "manual_dexterity": "体力劳动",
        "physics": "物理",
        "chemistry": "化学",
        "teaching": "教学",
        "math": "数学",
        "mathematics": "数学"
    },
    "en": {
        "观察与共情": "observation_and_empathy",
        "观察力": "observation",
        "创造力": "creativity",
        "吉他演奏": "guitar_playing",
        "歌词创作": "lyrics_creation",
        "人际沟通": "interpersonal_expression",
        "烹饪": "cooking",
        "烹饪基础": "cooking_basics",
        "体力劳动": "manual_dexterity",
        "物理": "physics",
        "化学": "chemistry",
        "教学": "teaching",
        "数学": "mathematics"
    }
}

def standardize_skill_key(key: str, language: str) -> str:
    """Standardize skill key based on the runtime language to avoid CN/EN split."""
    normalized_key = key.strip().lower()
    
    # 获取目标语言的转换字典
    lang_map = SKILL_TRANSLATION_MAP.get(language, {})
    
    # 尝试映射转换
    if normalized_key in lang_map:
        return lang_map[normalized_key]
        
    # 如果没找到映射，尝试原始 key 与字典的原样匹配
    for orig_key, mapped_key in lang_map.items():
        if orig_key.lower() == normalized_key:
            return mapped_key
            
    return key
```

#### 2.2 修改位置 2：在所有合并/保存 `skills` 字典的入口，强制应用清洗

在 `src/agents/data_manager.py`（以及 `src/world/activity.py` 的数值合并逻辑）中，更新 `save_state` 和 `update_state` 环节：

```python
    def standardize_and_merge_skills(self, current_skills: Dict[str, int], delta_skills: Dict[str, int]) -> Dict[str, int]:
        """Merge delta_skills into current_skills with standardization applied."""
        from src.utils import standardize_skill_key
        lang = self.language  # "zh" 或 "en"
        
        merged = dict(current_skills)
        for k, v in delta_skills.items():
            std_k = standardize_skill_key(k, lang)
            merged[std_k] = merged.get(std_k, 0) + v
        return merged
```

---

## 3. 归还私有灵魂：解绑 judge_others 阶段的 God Model 强制格式化

### 【诊断结论】
在 Agent 对周围邻居/同学进行“喜爱（Affection）”与“尊重（Respect）”的主观声誉评判时，代码强制将 Agent 自身的 `role_model` 替换为绝对中立的 `god_model`。这抹杀了 Agent 基于自身个性和独特阶级意识产生的爱恨偏见，使社交代币沦为高度同质化、去人格化的数值工具。

### 【重构方案】
在 `src/agents/role_agent.py` 的 `judge_others()` 方法中，解绑强制覆盖。允许使用 Agent 自己的大脑（`self.model`）来进行打分，同时在配置中留下开关（`enable_god_judging_override`）供科研对齐。

#### 3.1 修改位置
`src/agents/role_agent.py` 约第 1757 行：

**修改前：**
```python
            # Force the call to use god_model to ensure we evaluate on a fair, uniform model scale
            res = self._generate_with_functions(
                inputs,
                max_rounds=5,
                no_write_funcs=True,
                save_to_week_response=False,
                keep_compact_reasoning=False,
                model_override=self.config["god_model"],
                format_validator=validator,
            )
```

**修改后：**
```python
            # Allow agents to judge others using their own brain (role_model) 
            # to preserve ideological biases, class consciousness, and personalized relationship evaluation.
            # Fallback to god_model only if explicitly configured in config.json.
            use_god_override = self.config.get("response_validation", {}).get("enable_god_judging_override", False)
            model_override_val = self.config["god_model"] if use_god_override else None
            
            res = self._generate_with_functions(
                inputs,
                max_rounds=5,
                no_write_funcs=True,
                save_to_week_response=False,
                keep_compact_reasoning=False,
                model_override=model_override_val,
                format_validator=validator,
            )
```

---

## 4. 解耦唯心主义：God Model 语义因果与匹配算法重构展望

### 【设计蓝图】
为了使 Agentopia 具有更强的“唯物历史演化”性质，在未来的迭代中，后续代理可按照以下思路解耦 God Model 的语义垄断：

1. **职位双向招录（Gale-Shapley 重构）**：
   * 废除 God Model 的一刀切招录。
   * 让 Agent 根据其真实技能（`skills`）和存款在底层对 Positions 提交偏好排行；职位组织根据职位要求（`min_skills`）对 Agent 进行客观计算和打分。
   * 使用 **Gale-Shapley 延迟接受匹配算法** 运行多边匹配，使职位的分配成为客观生产力竞争的结果，而非主宰大模型的语义随机摇号。

2. **状态变动的物理/经济因果公式化**：
   * 日常 Solo/Joint 动作的 `delta_vitality` 与 `delta_fulfillment` 应该逐步引入基准公式。例如：
     $$\Delta \text{vitality} = -(\text{劳动强度系数}) + \text{摄入卡路里/睡眠折算}$$
     $$\Delta \text{material} = \text{生活资料消费价值} \times \text{边际消费效用}$$
   * 语言（大模型输出）仅作为对该物理/经济结果在“上层建筑”中的体验描述、情感表达与叙事修饰，而不应作为现实因果反馈的第一推动力。
