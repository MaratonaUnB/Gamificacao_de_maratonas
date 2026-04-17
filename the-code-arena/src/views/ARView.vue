<template>
    <section class="ar-page" :class="{ 'is-ready': arPronto }">
        <div v-if="arErro" class="ar-erro">{{ arErro }}</div>
        <div v-if="!arErro && !arPronto" class="ar-loader">Iniciando camera...</div>

        <a-scene v-if="arIniciado" embedded class="ar-scene"
            style="position: fixed; inset: 0; width: 100%; height: 100%; display: block; overflow: hidden; background: transparent;"
            renderer="alpha: true; antialias: true; precision: medium; logarithmicDepthBuffer: true;" :arjs="arJsConfig"
            device-orientation-permission-ui="enabled: false" vr-mode-ui="enabled: false" @loaded="onSceneLoaded">
            <a-nft ref="nftEl" type="nft" :url="markerArUrl" smooth="true" smoothCount="5"
                smoothTolerance="0.01" smoothThreshold="2">
                <!-- Aumentado a escala para visibilidade em NFT -->
                <a-entity :gltf-model="`url(${treeModelUrl})`" scale="100 100 100" position="0 0 0" rotation="-90 0 0"></a-entity>
            </a-nft>

            <a-entity camera></a-entity>
        </a-scene>
    </section>
    <button @click.stop.prevent="voltarParaLanding" @touchstart.stop="voltarParaLanding" class="btn-voltar">
        Voltar para a Arena
    </button>
</template>

<script setup>
import { nextTick, onMounted, onBeforeUnmount, ref } from 'vue';
import { useRouter } from 'vue-router';

const arIniciado = ref(false);
const arPronto = ref(false);
const arErro = ref('');
const arJsConfig = ref('trackingMethod: nft; sourceType: webcam; debugUIEnabled: false;');

const resolverBasePublica = () => {
    const baseEnv = import.meta.env.BASE_URL || '/';
    const comBarraInicial = baseEnv.startsWith('/') ? baseEnv : `/${baseEnv}`;
    return comBarraInicial.endsWith('/') ? comBarraInicial : `${comBarraInicial}/`;
};

const montarUrlPublica = (caminhoRelativo) => {
    const base = resolverBasePublica();
    const caminho = caminhoRelativo.replace(/^\/+/, '');
    return `${window.location.origin}${base}${caminho}`;
};

const markerArUrl = montarUrlPublica('image/trex');
const treeModelUrl = montarUrlPublica('image/Disco3d-1.glb');
let htmlClasseAnterior = '';
let bodyClasseAnterior = '';
let htmlOverflowAnterior = '';
let bodyOverflowAnterior = '';
let nftMarkerFoundHandler = null;
let nftMarkerLostHandler = null;
let resizeLogTimer = null;
let enforceLayoutInterval = null;
const nftEl = ref(null);
const router = useRouter();

const montarArJsConfig = () => {
    arJsConfig.value = 'trackingMethod: nft; sourceType: webcam; debugUIEnabled: false;';
};

const encerrarAR = () => {
    console.log('[AR DEBUG] Encerrando AR...');
    try {
        arIniciado.value = false;
        arPronto.value = false;

        const videos = document.querySelectorAll('video');
        videos.forEach((video) => {
            try {
                const stream = video.srcObject;
                if (stream && typeof stream.getTracks === 'function') {
                    stream.getTracks().forEach((track) => track.stop());
                    video.srcObject = null;
                }
                video.pause();
            } catch (e) {
                console.warn('[AR DEBUG] Erro ao parar video:', e);
            }
        });

        const seletoresAR = [
            'video.arjs-video',
            '#arjs-video',
            'canvas.a-canvas',
            '.a-canvas',
            '.arjs-loader',
            '.a-enter-vr',
            '.a-orientation-modal',
            '.a-loader-title',
            'a-scene[arjs]'
        ];

        seletoresAR.forEach((seletor) => {
            try {
                document.querySelectorAll(seletor).forEach((el) => el.remove());
            } catch (e) {
                console.warn(`[AR DEBUG] Erro ao remover elemento ${seletor}:`, e);
            }
        });

        document.documentElement.classList.remove('ar-mode');
        document.body.classList.remove('ar-mode');
        document.getElementById('app')?.classList.remove('ar-mode');
    } catch (e) {
        console.error('[AR DEBUG] Erro critico ao encerrar AR:', e);
    }
};

const aplicarLayoutTelaCheia = () => {
    const vv = window.visualViewport;
    const largura = `${Math.round(vv?.width ?? window.innerWidth)}px`;
    const altura = `${Math.round(vv?.height ?? window.innerHeight)}px`;

    document.documentElement.style.setProperty('--ar-vw', largura);
    document.documentElement.style.setProperty('--ar-vh', altura);

    const seletores = [
        'a-scene.ar-scene',
        '#arjs-video',
        'video.arjs-video',
        'video',
        'canvas',
        '.a-canvas'
    ];

    seletores.forEach((seletor) => {
        document.querySelectorAll(seletor).forEach((el) => {
            el.style.position = 'fixed';
            el.style.top = '0';
            el.style.left = '0';
            el.style.right = 'auto';
            el.style.bottom = 'auto';
            el.style.width = largura;
            el.style.height = altura;
            el.style.maxWidth = largura;
            el.style.maxHeight = altura;
            el.style.minWidth = largura;
            el.style.minHeight = altura;
            el.style.transform = 'none';
            el.style.margin = '0';
            el.style.padding = '0';
        });
    });

    document.querySelectorAll('video, #arjs-video, .arjs-video').forEach((el) => {
        el.style.setProperty('z-index', '10001', 'important');
    });
    document.querySelectorAll('canvas, .a-canvas').forEach((el) => {
        el.style.setProperty('z-index', '10002', 'important');
    });
    document.querySelectorAll('a-scene.ar-scene').forEach((el) => {
        el.style.setProperty('z-index', '10000', 'important');
    });
    document.querySelectorAll('.btn-voltar').forEach((el) => {
        el.style.setProperty('z-index', '2147483647', 'important');
    });
};

const logArLayout = (origem) => {
    const scene = document.querySelector('a-scene.ar-scene');
    const video = document.querySelector('#arjs-video, video.arjs-video, video');
    const canvas = document.querySelector('.a-canvas, canvas');

    const obter = (el) => {
        if (!el) return null;
        const rect = el.getBoundingClientRect();
        return {
            tag: el.tagName?.toLowerCase(),
            largura: Math.round(rect.width),
            altura: Math.round(rect.height),
            top: Math.round(rect.top),
            left: Math.round(rect.left)
        };
    };

    console.log('[AR DEBUG]', {
        origem,
        viewport: { largura: window.innerWidth, altura: window.innerHeight },
        markerArUrl,
        modeloUrl: treeModelUrl,
        scene: obter(scene),
        video: obter(video),
        canvas: obter(canvas)
    });
};

const validarAsset = async (url) => {
    try {
        const resposta = await fetch(url, { method: 'HEAD' });
        return resposta.ok;
    } catch {
        return false;
    }
};

const validarArquivosAR = async () => {
    const prefixo = markerArUrl;
    const checagens = [
        { url: `${prefixo}.iset`, desc: 'Descriptor ISET' },
        { url: `${prefixo}.fset`, desc: 'Descriptor FSET' },
        { url: `${prefixo}.fset3`, desc: 'Descriptor FSET3' },
        { url: treeModelUrl, desc: 'Modelo 3D (GLB)' }
    ];

    console.log('[AR DEBUG] Verificando integridade dos assets...');
    const resultados = await Promise.all(checagens.map(async (item) => {
        const existe = await validarAsset(item.url);
        return { ...item, existe };
    }));

    console.table(resultados);

    const falhas = resultados.filter(r => !r.existe);
    if (falhas.length > 0) {
        console.error('[AR DEBUG] Falha ao carregar os seguintes assets:', falhas);
    } else {
        console.log('[AR DEBUG] Todos os assets validados com sucesso.');
    }

    return falhas.length === 0;
};

const onResize = () => {
    aplicarLayoutTelaCheia();
    if (resizeLogTimer) clearTimeout(resizeLogTimer);
    resizeLogTimer = setTimeout(() => logArLayout('resize'), 120);
};

const onSceneLoaded = () => {
    arPronto.value = true;
    aplicarLayoutTelaCheia();
    logArLayout('scene-loaded');
};

const registrarEventosMarker = () => {
    const el = nftEl.value;
    if (!el) return;

    nftMarkerFoundHandler = () => {
        console.log('[AR DEBUG] Marker encontrado: botonCerrado');
        logArLayout('marker-found');
    };
    nftMarkerLostHandler = () => {
        console.log('[AR DEBUG] Marker perdido: botonCerrado');
    };

    el.addEventListener('markerFound', nftMarkerFoundHandler);
    el.addEventListener('markerLost', nftMarkerLostHandler);

    el.addEventListener('arjs-nft-loaded', (ev) => {
        console.log('[AR DEBUG] NFT Data Loaded:', ev.detail);
    });
};

const removerEventosMarker = () => {
    const el = nftEl.value;
    if (!el) return;
    if (nftMarkerFoundHandler) el.removeEventListener('markerFound', nftMarkerFoundHandler);
    if (nftMarkerLostHandler) el.removeEventListener('markerLost', nftMarkerLostHandler);
    nftMarkerFoundHandler = null;
    nftMarkerLostHandler = null;
};

const iniciarAR = async () => {
    arErro.value = '';
    montarArJsConfig();
    console.log('[AR DEBUG] Iniciando AR para marker botonCerrado');

    if (!window.isSecureContext) {
        arErro.value = 'Para usar a camera, abra o projeto via localhost (Vite) ou HTTPS.';
        return;
    }

    if (!navigator.mediaDevices?.getUserMedia) {
        arErro.value = 'Seu navegador nao suporta acesso a camera.';
        return;
    }

    const arquivosOk = await validarArquivosAR();
    if (!arquivosOk) {
        arErro.value = 'Arquivos do marker botonCerrado/modelo nao encontrados. Verifique o console.';
        return;
    }

    try {
        const stream = await navigator.mediaDevices.getUserMedia({
            video: { facingMode: 'environment' },
            audio: false
        });
        stream.getTracks().forEach((track) => track.stop());

        arIniciado.value = true;
        await nextTick();
        registrarEventosMarker();
        aplicarLayoutTelaCheia();
        logArLayout('ar-started');
    } catch {
        arErro.value = 'Permissao da camera bloqueada. Libere a camera nas configuracoes do navegador.';
    }
};

const voltarParaLanding = () => {
    console.log('[AR DEBUG] Botão voltar clicado');
    encerrarAR();
    setTimeout(() => {
        router.replace({ name: 'landing' });
    }, 50);
};

onMounted(() => {
    htmlClasseAnterior = document.documentElement.className;
    bodyClasseAnterior = document.body.className;
    htmlOverflowAnterior = document.documentElement.style.overflow;
    bodyOverflowAnterior = document.body.style.overflow;

    document.documentElement.classList.add('ar-mode');
    document.body.classList.add('ar-mode');
    document.getElementById('app')?.classList.add('ar-mode');

    document.documentElement.style.overflow = 'hidden';
    document.body.style.overflow = 'hidden';

    window.addEventListener('resize', onResize, { passive: true });
    window.visualViewport?.addEventListener('resize', onResize, { passive: true });
    window.visualViewport?.addEventListener('scroll', onResize, { passive: true });
    enforceLayoutInterval = window.setInterval(aplicarLayoutTelaCheia, 400);
    iniciarAR();
    nextTick(() => {
        aplicarLayoutTelaCheia();
        logArLayout('mounted');
    });
});

onBeforeUnmount(() => {
    encerrarAR();
    removerEventosMarker();
    window.removeEventListener('resize', onResize);
    window.visualViewport?.removeEventListener('resize', onResize);
    window.visualViewport?.removeEventListener('scroll', onResize);
    if (resizeLogTimer) clearTimeout(resizeLogTimer);
    if (enforceLayoutInterval) clearInterval(enforceLayoutInterval);
    document.documentElement.style.overflow = htmlOverflowAnterior;
    document.body.style.overflow = bodyOverflowAnterior;
    document.documentElement.classList.remove('ar-mode');
    document.body.classList.remove('ar-mode');
    document.getElementById('app')?.classList.remove('ar-mode');
    document.documentElement.className = htmlClasseAnterior;
    document.body.className = bodyClasseAnterior;
    window.scrollTo({ top: 0, left: 0, behavior: 'auto' });
});
</script>

<style scoped>
.ar-page {
    position: fixed;
    inset: 0;
    width: var(--ar-vw, 100vw);
    height: var(--ar-vh, 100vh);
    z-index: 9999;
    background: #000;
    overflow: hidden;
}

.ar-page.is-ready {
    background: #000;
}

.ar-scene {
    position: fixed;
    inset: 0;
    width: var(--ar-vw, 100vw) !important;
    height: var(--ar-vh, 100vh) !important;
    display: block;
    background: transparent !important;
    z-index: 10000 !important;
    pointer-events: none;
}

.btn-voltar {
    position: fixed;
    top: 20px;
    right: 20px;
    z-index: 2147483647 !important;
    pointer-events: auto !important;
    user-select: none;
    -webkit-tap-highlight-color: transparent;
    padding: 10px 20px;
    background: #ff4444;
    color: #fff;
    border: none;
    font-family: "VT323", monospace;
    font-size: 1.25rem;
    cursor: pointer;
    pointer-events: auto;
}

.ar-loader {
    position: fixed;
    inset: 0;
    z-index: 10004;
    display: grid;
    place-items: center;
    color: #fff;
    font-family: "VT323", monospace;
    font-size: 1.35rem;
    background: rgba(0, 0, 0, 0.7);
}

.ar-erro {
    position: fixed;
    left: 50%;
    top: 50%;
    transform: translate(-50%, -50%);
    z-index: 10004;
    color: #ff8080;
    text-align: center;
    font-size: 1.1rem;
    max-width: min(90vw, 560px);
}

:global(video),
:global(#arjs-video),
:global(.a-canvas),
:global(canvas),
:global(.arjs-video) {
    position: fixed !important;
    top: 0 !important;
    left: 0 !important;
    right: auto !important;
    bottom: auto !important;
    width: var(--ar-vw, 100vw) !important;
    height: var(--ar-vh, 100vh) !important;
    min-width: var(--ar-vw, 100vw) !important;
    min-height: var(--ar-vh, 100vh) !important;
    max-width: var(--ar-vw, 100vw) !important;
    max-height: var(--ar-vh, 100vh) !important;
    margin: 0 !important;
    padding: 0 !important;
    transform: none !important;
    background: transparent !important;
}

:global(a-scene.ar-scene),
:global(a-scene.ar-scene canvas),
:global(a-scene.ar-scene video) {
    position: fixed !important;
    top: 0 !important;
    left: 0 !important;
    right: auto !important;
    bottom: auto !important;
    width: var(--ar-vw, 100vw) !important;
    height: var(--ar-vh, 100vh) !important;
    min-width: var(--ar-vw, 100vw) !important;
    min-height: var(--ar-vh, 100vh) !important;
    transform: none !important;
    background: transparent !important;
}

:global(html.ar-mode),
:global(body.ar-mode),
:global(#app.ar-mode) {
    width: 100% !important;
    height: 100% !important;
    min-height: 100% !important;
    margin: 0 !important;
    overflow: hidden !important;
    background: transparent !important;
}

:global(video),
:global(#arjs-video),
:global(.arjs-video) {
    object-fit: cover !important;
    z-index: 10001 !important;
}

:global(.a-canvas),
:global(canvas) {
    z-index: 10002 !important;
}
</style>
