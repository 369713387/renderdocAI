# RenderDoc AI 插件开发文档

## 1. 项目概述

基于 RenderDoc PerfExport JSON 的 AI 辅助性能分析插件，提供深度 GPU 性能洞察和自动化优化建议。

### 1.1 核心功能
- **数据解析**：解析 RenderDoc PerfExport v2 格式
- **性能分析**：自动识别性能瓶颈
- **可视化**：GPU 时间线、资源占用图表
- **AI 建议**：基于规则的优化建议生成

---

## 2. 架构设计

```
┌─────────────────────────────────────────────────────────────┐
│                    RenderDoc AI Plugin                       │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │ Data Parser │  │  Analyzer   │  │ Visualizer  │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │ AI Engine   │  │ Report Gen  │  │ Export Tool │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. 核心模块

### 3.1 数据解析器 (DataParser)

```python
class PerfExportParser:
    """RenderDoc PerfExport JSON 解析器"""
    
    def __init__(self, json_path: str):
        self.data = self._load_json(json_path)
        self.confidence_legend = self.data.get('confidence_legend', {})
    
    def parse_meta(self) -> MetaInfo:
        """解析元数据"""
        meta = self.data['meta']
        return MetaInfo(
            gpu=GPUInfo(**meta['gpu']),
            resolution=Resolution(**meta['resolution']),
            frame_stats=FrameStats(**meta['frame_stats']),
            threshold_ms=meta['threshold_ms']
        )
    
    def parse_passes(self) -> List[RenderPass]:
        """解析所有渲染 Pass"""
        passes = []
        for pass_data in self.data.get('passes', []):
            render_pass = RenderPass(
                pass_index=pass_data['pass_index'],
                pass_name=pass_data['pass_name'],
                timing=TimingInfo(**pass_data['timing']),
                render_targets=self._parse_render_targets(pass_data.get('render_targets', {})),
                draws=self._parse_draws(pass_data.get('draws', []))
            )
            passes.append(render_pass)
        return passes
    
    def parse_resource_inventory(self) -> ResourceInventory:
        """解析资源清单"""
        inv = self.data.get('resource_inventory', {})
        return ResourceInventory(
            render_targets=[RenderTarget(**rt) for rt in inv.get('render_targets', [])],
            total_rt_memory_mb=inv.get('total_rt_memory_mb', 0)
        )
```

### 3.2 性能分析器 (Analyzer)

```python
class PerformanceAnalyzer:
    """性能分析引擎"""
    
    def __init__(self, parser: PerfExportParser):
        self.parser = parser
        self.meta = parser.parse_meta()
        self.passes = parser.parse_passes()
        self.resources = parser.parse_resource_inventory()
    
    def analyze_bottlenecks(self) -> List[Bottleneck]:
        """识别性能瓶颈"""
        bottlenecks = []
        
        # 1. 高耗时 Pass 检测
        for pass_ in self.passes:
            timing = pass_.timing
            if timing.gpu_time_ms > self.meta.threshold_ms:
                bottlenecks.append(Bottleneck(
                    type=BottleneckType.HIGH_GPU_TIME,
                    pass_name=pass_.pass_name,
                    severity=self._calculate_severity(timing.gpu_time_ms),
                    details=f"GPU time: {timing.gpu_time_ms}ms ({timing.gpu_time_percentage}%)"
                ))
        
        # 2. 过度绘制检测
        for pass_ in self.passes:
            if hasattr(pass_, 'gpu_counters'):
                overdraw = pass_.gpu_counters.get('overdraw_ratio', 1.0)
                if overdraw > 2.0:
                    bottlenecks.append(Bottleneck(
                        type=BottleneckType.OVERDRAW,
                        pass_name=pass_.pass_name,
                        severity=Severity.MEDIUM,
                        details=f"Overdraw ratio: {overdraw:.2f}x"
                    ))
        
        # 3. 带宽瓶颈检测
        for pass_ in self.passes:
            if hasattr(pass_, 'bandwidth_estimate'):
                bw = pass_.bandwidth_estimate
                total_mb = bw.get('total_estimated_mb', 0)
                if total_mb > 1000:  # > 1GB
                    bottlenecks.append(Bottleneck(
                        type=BottleneckType.HIGH_BANDWIDTH,
                        pass_name=pass_.pass_name,
                        severity=Severity.HIGH,
                        details=f"Estimated bandwidth: {total_mb:.1f} MB"
                    ))
        
        return sorted(bottlenecks, key=lambda x: x.severity.value, reverse=True)
    
    def generate_optimization_suggestions(self) -> List[Suggestion]:
        """生成优化建议"""
        suggestions = []
        bottlenecks = self.analyze_bottlenecks()
        
        for bottleneck in bottlenecks:
            if bottleneck.type == BottleneckType.HIGH_GPU_TIME:
                if "GBuffer" in bottleneck.pass_name:
                    suggestions.append(Suggestion(
                        category="Geometry",
                        priority=Priority.HIGH,
                        description="GBuffer 阶段耗时过高，考虑 LOD 优化",
                        actions=[
                            "启用动态 LOD 系统",
                            "降低远距离物体面数",
                            "使用遮挡剔除剔除不可见物体"
                        ]
                    ))
                elif "Lighting" in bottleneck.pass_name:
                    suggestions.append(Suggestion(
                        category="Shading",
                        priority=Priority.HIGH,
                        description="光照阶段耗时过高，优化着色复杂度",
                        actions=[
                            "降低光源数量或合并光源",
                            "使用 tiled/clustered deferred 替代 full-screen",
                            "优化 BRDF 计算，考虑 LUT 近似"
                        ]
                    ))
                elif "Shadow" in bottleneck.pass_name:
                    suggestions.append(Suggestion(
                        category="Shadow",
                        priority=Priority.MEDIUM,
                        description="阴影渲染耗时高，优化 CSM 设置",
                        actions=[
                            "减少 cascade 数量",
                            "降低 shadow map 分辨率",
                            "使用 PCSS 替代 PCF",
                            "启用 cascade 缓存"
                        ]
                    ))
            
            elif bottleneck.type == BottleneckType.OVERDRAW:
                suggestions.append(Suggestion(
                    category="Overdraw",
                    priority=Priority.HIGH,
                    description="存在严重过度绘制",
                    actions=[
                        "启用 Early-Z/Hi-Z",
                        "从前向后排序不透明物体",
                        "使用遮挡查询剔除",
                        "考虑使用 depth pre-pass"
                    ]
                ))
            
            elif bottleneck.type == BottleneckType.HIGH_BANDWIDTH:
                suggestions.append(Suggestion(
                    category="Bandwidth",
                    priority=Priority.MEDIUM,
                    description="显存带宽占用过高",
                    actions=[
                        "启用纹理压缩 (BC7/BC5)",
                        "使用 mipmapping 减少纹理采样",
                        "合并 render target 减少带宽",
                        "考虑使用 tiled rendering"
                    ]
                ))
        
        return suggestions
```

### 3.3 可视化模块 (Visualizer)

```python
class PerformanceVisualizer:
    """性能数据可视化"""
    
    def __init__(self, analyzer: PerformanceAnalyzer):
        self.analyzer = analyzer
    
    def generate_gpu_timeline(self) -> TimelineChart:
        """生成 GPU 时间线"""
        timeline = TimelineChart(title="GPU Frame Timeline")
        
        for pass_ in self.analyzer.passes:
            timing = pass_.timing
            timeline.add_bar(
                name=pass_.pass_name,
                start=0,
                duration=timing.gpu_time_ms,
                color=self._get_pass_color(pass_.pass_name),
                percentage=timing.gpu_time_percentage
            )
        
        return timeline
    
    def generate_resource_usage_chart(self) -> PieChart:
        """生成资源使用饼图"""
        chart = PieChart(title="Render Target Memory Usage")
        
        for rt in self.analyzer.resources.render_targets:
            chart.add_slice(
                label=rt.name,
                value=rt.size_bytes / (1024 * 1024),  # MB
                color=self._get_rt_color(rt.name)
            )
        
        return chart
    
    def generate_bandwidth_chart(self) -> BarChart:
        """生成带宽使用柱状图"""
        chart = BarChart(title="Estimated Bandwidth per Pass")
        
        for pass_ in self.analyzer.passes:
            if hasattr(pass_, 'bandwidth_estimate'):
                bw = pass_.bandwidth_estimate
                chart.add_bar(
                    label=pass_.pass_name,
                    read=bw.get('rt_read_mb', 0) + bw.get('texture_read_mb', 0),
                    write=bw.get('rt_write_mb', 0)
                )
        
        return chart
```

---

## 4. AI 优化引擎

### 4.1 规则引擎

```python
class OptimizationRuleEngine:
    """基于规则的优化建议生成"""
    
    RULES = [
        # GPU 时间相关规则
        Rule(
            name="high_gbuffer_time",
            condition=lambda p: "GBuffer" in p.pass_name and p.timing.gpu_time_ms > 2.0,
            actions=[
                "启用动态 LOD 系统",
                "使用遮挡剔除",
                "降低远距离物体面数"
            ]
        ),
        
        # 过度绘制规则
        Rule(
            name="excessive_overdraw",
            condition=lambda p: p.gpu_counters.get('overdraw_ratio', 1.0) > 3.0,
            actions=[
                "启用 Early-Z/Hi-Z",
                "从前向后排序不透明物体",
                "使用 depth pre-pass"
            ]
        ),
        
        # 带宽规则
        Rule(
            name="high_bandwidth",
            condition=lambda p: p.bandwidth_estimate.get('total_estimated_mb', 0) > 500,
            actions=[
                "启用纹理压缩",
                "使用 mipmapping",
                "合并 render target"
            ]
        ),
        
        # 阴影规则
        Rule(
            name="expensive_shadows",
            condition=lambda p: "Shadow" in p.pass_name and p.timing.gpu_time_ms > 1.5,
            actions=[
                "减少 cascade 数量",
                "降低 shadow map 分辨率",
                "启用 cascade 缓存"
            ]
        ),
    ]
    
    def evaluate(self, render_pass: RenderPass) -> List[Suggestion]:
        """评估渲染 pass 并生成建议"""
        suggestions = []
        
        for rule in self.RULES:
            if rule.condition(render_pass):
                suggestions.append(Suggestion(
                    rule=rule.name,
                    pass_name=render_pass.pass_name,
                    actions=rule.actions,
                    severity=self._calculate_severity(render_pass, rule)
                ))
        
        return suggestions
```

### 4.2 机器学习模块（扩展）

```python
class MLPerformancePredictor:
    """基于历史数据的性能预测"""
    
    def __init__(self):
        self.model = None  # 预训练的回归模型
    
    def predict_frame_time(self, scene_complexity: SceneComplexity) -> float:
        """预测帧时间"""
        features = self._extract_features(scene_complexity)
        return self.model.predict(features)
    
    def recommend_settings(self, target_fps: int) -> RenderSettings:
        """推荐渲染设置以达到目标帧率"""
        # 基于性能模型反推最优设置
        pass
```

---

## 5. API 接口

### 5.1 Python SDK

```python
from renderdoc_ai import PerfAnalyzer

# 加载性能数据
analyzer = PerfAnalyzer.from_json("renderdoc_perf_export.json")

# 获取性能概览
summary = analyzer.get_summary()
print(f"Total GPU time: {summary.gpu_time_ms}ms")
print(f"Draw calls: {summary.draw_calls}")

# 分析瓶颈
bottlenecks = analyzer.analyze_bottlenecks()
for b in bottlenecks:
    print(f"[{b.severity}] {b.type}: {b.details}")

# 获取优化建议
suggestions = analyzer.generate_suggestions()
for s in suggestions:
    print(f"\n{s.category} ({s.priority}):")
    print(f"  {s.description}")
    for action in s.actions:
        print(f"  - {action}")

# 生成可视化图表
analyzer.generate_timeline_chart().save("timeline.png")
analyzer.generate_resource_chart().save("resources.png")
```

### 5.2 CLI 工具

```bash
# 分析性能文件
renderdoc-ai analyze renderdoc_perf_export.json

# 生成详细报告
renderdoc-ai analyze renderdoc_perf_export.json --format html --output report.html

# 导出瓶颈列表
renderdoc-ai analyze renderdoc_perf_export.json --bottlenecks-only --format json

# 比较两个性能文件
renderdoc-ai compare before.json after.json
```

### 5.3 REST API

```python
from flask import Flask, request, jsonify

app = Flask(__name__)

@app.route('/api/v1/analyze', methods=['POST'])
def analyze():
    """上传 JSON 文件进行分析"""
    file = request.files['file']
    analyzer = PerfAnalyzer.from_json(file)
    
    return jsonify({
        'summary': analyzer.get_summary().to_dict(),
        'bottlenecks': [b.to_dict() for b in analyzer.analyze_bottlenecks()],
        'suggestions': [s.to_dict() for s in analyzer.generate_suggestions()]
    })

@app.route('/api/v1/visualize/timeline', methods=['POST'])
def timeline():
    """生成 GPU 时间线数据"""
    file = request.files['file']
    analyzer = PerfAnalyzer.from_json(file)
    chart = analyzer.generate_timeline_chart()
    return jsonify(chart.to_dict())
```

---

## 6. 开发指南

### 6.1 环境搭建

```bash
# 克隆仓库
git clone https://github.com/369713387/renderdocAI.git
cd renderdocAI

# 创建虚拟环境
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# 安装依赖
pip install -r requirements.txt

# 安装开发依赖
pip install -r requirements-dev.txt
```

### 6.2 项目结构

```
renderdocAI/
├── src/
│   ├── renderdoc_ai/
│   │   ├── __init__.py
│   │   ├── parser.py          # JSON 解析器
│   │   ├── analyzer.py        # 性能分析器
│   │   ├── visualizer.py      # 可视化模块
│   │   ├── rules.py           # 优化规则引擎
│   │   └── api.py             # API 接口
│   └── cli/
│       └── renderdoc-ai.py    # CLI 工具
├── tests/
│   ├── test_parser.py
│   ├── test_analyzer.py
│   └── fixtures/
│       └── sample_export.json
├── docs/
│   ├── api.md
│   └── development.md
├── requirements.txt
├── requirements-dev.txt
└── setup.py
```

### 6.3 添加新规则

```python
# src/renderdoc_ai/rules.py

class CustomRule(Rule):
    """自定义优化规则示例"""
    
    def __init__(self):
        super().__init__(
            name="custom_texture_optimization",
            description="检测大尺寸未压缩纹理"
        )
    
    def evaluate(self, context: RenderContext) -> List[Finding]:
        findings = []
        
        for texture in context.textures:
            if texture.size > 2048 and not texture.is_compressed:
                findings.append(Finding(
                    rule=self.name,
                    severity=Severity.MEDIUM,
                    message=f"Texture '{texture.name}' ({texture.size}x{texture.size}) should be compressed",
                    actions=[
                        f"Compress {texture.name} using BC7 or ASTC",
                        "Generate mipmaps if not present"
                    ]
                ))
        
        return findings

# 注册规则
RuleRegistry.register(CustomRule())
```

### 6.4 运行测试

```bash
# 运行所有测试
pytest

# 运行特定测试
pytest tests/test_analyzer.py -v

# 生成覆盖率报告
pytest --cov=renderdoc_ai --cov-report=html

# 性能测试
pytest tests/performance/ --benchmark-only
```

---

## 7. 路线图

### v1.0 - MVP
- [x] JSON 解析器
- [x] 基础性能分析
- [x] 规则引擎
- [ ] CLI 工具
- [ ] 基础可视化

### v1.1
- [ ] 高级瓶颈检测算法
- [ ] 多帧对比分析
- [ ] 更多可视化图表
- [ ] Python SDK 完善

### v2.0
- [ ] AI 模型集成（性能预测）
- [ ] 自动优化建议生成
- [ ] REST API 服务
- [ ] Web 可视化界面
- [ ] 多平台支持（Linux/macOS）

---

## 8. 贡献指南

1. Fork 仓库
2. 创建功能分支 (`git checkout -b feature/amazing-feature`)
3. 提交更改 (`git commit -m 'Add amazing feature'`)
4. 推送到分支 (`git push origin feature/amazing-feature`)
5. 创建 Pull Request

---

## 9. 许可证

MIT License - 详见 [LICENSE](LICENSE)

---

## 10. 参考资料

- [RenderDoc Documentation](https://renderdoc.org/docs/)
- [Vulkan Performance Guide](https://developer.nvidia.com/vulkan-performance-guide)
- [GPU Performance Analysis](https://gpuopen.com/performance/)

---

*文档版本: 1.0*
*最后更新: 2026-03-12*
