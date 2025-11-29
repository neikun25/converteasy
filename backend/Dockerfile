# syntax=docker/dockerfile:1.4
##############################################
# ConvertEasy Backend - 多阶段构建优化版
# 支持文档转换（PDF/Word/Excel/PPT）和音频转换
##############################################

# 构建参数
ARG NODE_VERSION=18
ARG PYTHON_VERSION=3.11

##############################################
# Stage 1: Node.js 构建阶段
##############################################
FROM node:${NODE_VERSION}-slim AS node-builder

WORKDIR /app

# 安装构建依赖并构建（单层合并）
RUN apt-get update && apt-get install -y --no-install-recommends \
    python3 make g++ \
    && rm -rf /var/lib/apt/lists/*

# 利用 Docker 缓存：先复制依赖文件
COPY package*.json tsconfig.json ./

# 安装依赖 + 构建 + 清理（单层）
RUN npm ci --ignore-scripts \
    && npm cache clean --force

# 复制源码并构建
COPY src ./src
RUN npm run build \
    && npm prune --production \
    && rm -rf src tsconfig.json

##############################################
# Stage 2: Python 依赖构建阶段
##############################################
FROM python:${PYTHON_VERSION}-slim AS python-builder

WORKDIR /app

# 安装编译依赖（单层）
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc g++ libcairo2-dev libffi-dev pkg-config \
    && rm -rf /var/lib/apt/lists/*

# 创建虚拟环境并安装依赖
COPY requirements.txt .
RUN python3 -m venv /opt/venv \
    && /opt/venv/bin/pip install --no-cache-dir -U pip wheel \
    && /opt/venv/bin/pip install --no-cache-dir -r requirements.txt \
    && find /opt/venv -type d -name "__pycache__" -exec rm -rf {} + 2>/dev/null || true \
    && find /opt/venv -type f -name "*.pyc" -delete 2>/dev/null || true

##############################################
# Stage 3: 运行时镜像（精简版）
##############################################
FROM node:${NODE_VERSION}-slim AS runtime

# 镜像元数据
LABEL maintainer="ConvertEasy Team" \
      version="1.0.0" \
      description="File conversion service with document and audio support"

# 环境变量
ENV NODE_ENV=production \
    PYTHON_PATH=/opt/venv/bin/python3 \
    PATH="/opt/venv/bin:$PATH" \
    # 禁用 Python 字节码缓存
    PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /app

# 安装运行时依赖（单层，移除冗余的 python3 python3-venv）
# 注：已从 python-builder 复制完整 venv，无需系统 Python
RUN apt-get update && apt-get install -y --no-install-recommends \
    # 基础依赖
    ca-certificates \
    curl \
    # Python C 扩展运行时库
    libcairo2 \
    libffi8 \
    libglib2.0-0 \
    # LibreOffice 无头模式（文档转换）
    libreoffice-core-nogui \
    libreoffice-writer-nogui \
    libreoffice-calc-nogui \
    libreoffice-impress-nogui \
    # 字体支持
    fonts-liberation \
    fonts-noto-cjk \
    # 音频转换
    ffmpeg \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/* \
    # 创建应用目录
    && mkdir -p /app/uploads /app/public \
    && chmod 755 /app/uploads /app/public

# 复制 Node.js 构建产物
COPY --from=node-builder /app/dist ./dist
COPY --from=node-builder /app/node_modules ./node_modules
COPY --from=node-builder /app/package.json ./

# 复制 Python 虚拟环境
COPY --from=python-builder /opt/venv /opt/venv

# 确保 python3 符号链接存在
RUN ln -sf /opt/venv/bin/python3.11 /opt/venv/bin/python3 2>/dev/null || true \
    && ln -sf /opt/venv/bin/python3.11 /opt/venv/bin/python 2>/dev/null || true

# 复制 Python 转换脚本
COPY src/scripts ./src/scripts

# 复制证书配置（K8s lifecycle hook）
COPY cert /app/cert
RUN set -e; \
    # 设置 initenv.sh 权限
    if [ -f /app/cert/initenv.sh ]; then \
        chmod +x /app/cert/initenv.sh; \
    fi; \
    # 预加载证书到系统信任库
    if [ -f /app/cert/certificate.crt ]; then \
        cp /app/cert/certificate.crt /usr/local/share/ca-certificates/ 2>/dev/null || true; \
        update-ca-certificates 2>/dev/null || true; \
    fi; \
    # 替换为轻量级 initenv（避免运行时 OOM）
    if [ -f /app/cert/initenv.sh ]; then \
        printf '%s\n' \
            '#!/bin/sh' \
            'set -e' \
            '[ -d /app/uploads ] || mkdir -p /app/uploads' \
            '[ -d /app/public ] || mkdir -p /app/public' \
            'echo "[initenv] ready"' \
            'exit 0' > /app/cert/initenv.sh; \
        chmod +x /app/cert/initenv.sh; \
    fi

# 健康检查（使用 curl 更轻量）
HEALTHCHECK --interval=30s --timeout=5s --start-period=30s --retries=3 \
    CMD curl -fsS http://localhost:8080/health || exit 1

EXPOSE 8080

# 启动命令
CMD ["node", "dist/index.js"]