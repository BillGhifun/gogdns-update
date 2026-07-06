上传apk包到目标系统中，执行以下命令安装
```
apk add gogdns_xxxx.apk
```

如果出现以下提示
```
ERROR: /gogdns.apk: UNTRUSTED signature
```
尝试使用
```
apk add --allow-untrusted gogdns_xxxx.apk
```

---

ImmortalWrt apk
请使用脚本来打包新版二进制程序或使用提供的打包好的文件(可能更新不及时)

复制以下内容为`apk-package-tool.ps1`

```sh
# ============================================================
# GOGDNS APK 批量打包 (固定版本号版)
# 修改下面的 $VERSION 变量后运行即可
# ============================================================

# ============================================================
# !! 发布时，请修改这里 !!
# 格式: MainVersion.MainVersionServer(保留原格式即可)
# 例如: 0.6.89.20260706110410
$VERSION = "0.6.89.20260706163504"
# ============================================================

$ErrorActionPreference = "Stop"
$scriptDir = Split-Path -Parent $MyInvocation.MyCommand.Path
$binDir = Join-Path $scriptDir ".."

Write-Host "============================================" -ForegroundColor Cyan
Write-Host "  GOGDNS APK 批量打包 (固定版本)" -ForegroundColor Cyan
Write-Host "============================================" -ForegroundColor Cyan
Write-Host "  当前版本: $VERSION" -ForegroundColor Cyan
Write-Host "============================================" -ForegroundColor Cyan
Write-Host ""

# 架构映射
$archMap = @{
    "386"   = "i386_pentium4"
    "amd64" = "x86_64"
    "arm64" = "aarch64_generic"
    "arm"   = "arm_cortex-a7_neon-vfpv4"
}

# 读取 nfpm 模板
$tpl = Get-Content (Join-Path $scriptDir "nfpm.yaml") -Raw

# 创建输出目录
$outDir = Join-Path $scriptDir "output"
New-Item -ItemType Directory -Force -Path $outDir | Out-Null

# 批量打包
$ok = 0; $skip = 0; $fail = 0
foreach ($suffix in $archMap.Keys) {
    $arch = $archMap[$suffix]
    $binFile = Join-Path $binDir "gogdns-linux-$suffix"
    $outName = "gogdns_${VERSION}_${arch}.apk"
    $outPath = Join-Path $outDir $outName

    Write-Host "[$suffix -> $arch]" -ForegroundColor Yellow

    if (-not (Test-Path $binFile)) {
        Write-Host "  [SKIP] 文件不存在" -ForegroundColor Yellow
        $skip++
        continue
    }

    try {
        # 复制二进制
        Copy-Item $binFile (Join-Path $scriptDir "gogdns") -Force

        # 替换生成配置
        $cfg = $tpl -replace 'arch:\s*"[^"]*"', "arch: `"$arch`""
        $cfg = $cfg -replace 'version:\s*"[^"]*"', "version: `"$VERSION`""
        $cfgFile = Join-Path $scriptDir "nfpm_$suffix.yaml"
        $cfg | Set-Content $cfgFile -Encoding UTF8

        # 打包
        nfpm.exe package --config $cfgFile --target $outPath 2>&1 | Out-Null

        if ($LASTEXITCODE -eq 0) {
            Write-Host "  [OK] $outName" -ForegroundColor Green
            $ok++
        } else {
            Write-Host "  [FAIL] 打包失败" -ForegroundColor Red
            $fail++
        }

        # 清理临时文件
        Remove-Item $cfgFile -Force -ErrorAction SilentlyContinue
        Remove-Item (Join-Path $scriptDir "gogdns") -Force -ErrorAction SilentlyContinue
    } catch {
        Write-Host "  [FAIL] $_" -ForegroundColor Red
        $fail++
    }
}

Write-Host ""
Write-Host "============================================" -ForegroundColor Cyan
Write-Host "  打包完成: 成功 $ok | 跳过 $skip | 失败 $fail" -ForegroundColor Cyan
Write-Host "  输出目录: $outDir" -ForegroundColor Cyan
Write-Host "============================================" -ForegroundColor Cyan
```

把你要打包的相应的原始二进制程序 `gogdns-linux-386`、`gogdns-linux-amd64`、`gogdns-linux-arm`、`gogdns-linux-arm64`放到与`ipk-package-tool.sh`同目录下，然后双击运行`build-apk-manual.ps1`
