[Cocos Store](https://store.cocos.com/app/detail/3936) 和 [【玉兔 | 图形学与游戏开发】高性能的玻璃杯渲染Shader | 折射+反射+厚度+材质纹路 | 零基础也能学_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV15a411D7wZ/?spm_id_from=888.80997.embed_other.whitelist&t=8.33324&bvid=BV15a411D7wZ&vd_source=08ce52dcf1d45b5807e4ad1b7e0821f4)

注意点：

1. 玻璃为半透明材质，所以需要以 transparent 方式渲染
2. 由于将 matCap 贴图的 r 通道当作输出颜色的 alpha 值，所以需要使用灰度贴图，越白的地方为高光越不透明，越黑的地方越透明