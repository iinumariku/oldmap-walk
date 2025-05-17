<script lang="ts">
	import maplibregl from 'maplibre-gl';
	import type { StyleSpecification } from 'maplibre-gl';  // これを追加！
	import { MapLibre } from 'svelte-maplibre-gl';
	import { onMount } from 'svelte';

	let map: maplibregl.Map | undefined = $state.raw();
	let currentMarker: maplibregl.Marker | undefined;
	let opacity = $state(0.7); // 透明度の初期値
	let currentPosition: { lat: number; lng: number } | null = $state(null);

	// ベースマップの種類を定義
	const basemaps = $state({
		gsi_std: '地理院標準地図',
		gsi_pale: '地理院淡色地図',
		osm: 'OSM'
	});

	// 現在選択中のベースマップ
	let currentBasemap = $state('gsi_std');

	// レイヤーの初期スタイルを設定するための関数を追加
	function updateStyle() {
		const gsiStyle: StyleSpecification = {
			version: 8,
			sources: {
				gsi_std: {
					type: 'raster',
					tiles: [
						'https://cyberjapandata.gsi.go.jp/xyz/std/{z}/{x}/{y}.png'
					],
					tileSize: 256,
					attribution: '国土地理院',
				},
				gsi_pale: {
					type: 'raster',
					tiles: [
						'https://cyberjapandata.gsi.go.jp/xyz/pale/{z}/{x}/{y}.png'
					],
					tileSize: 256,
					attribution: '国土地理院',
				},
				osm: {
					type: 'raster',
					tiles: [
						'https://tile.openstreetmap.org/{z}/{x}/{y}.png'
					],
					tileSize: 256,
					attribution: '© OpenStreetMap contributors',
				},
				konjaku: {
					type: 'raster',
					tiles: [
						'https://ktgis.net/kjmapw/kjtilemap/keihansin/00/{z}/{x}/{y}.png'  // あなたの今昔マップのURLに変更してね
					],
					tileSize: 256,
					attribution: '今昔マップ京阪神圏1922-23',
					scheme: 'tms',
				}
			},
			layers: [
				{
					id: 'gsi-std-layer',
					type: 'raster',
					source: 'gsi_std',
					layout: {
						visibility: currentBasemap === 'gsi_std' ? 'visible' : 'none'
					},
					minzoom: 0,
					maxzoom: 18
				},
				{
					id: 'gsi-pale-layer',
					type: 'raster',
					source: 'gsi_pale',
					layout: {
						visibility: currentBasemap === 'gsi_pale' ? 'visible' : 'none'
					},
					minzoom: 0,
					maxzoom: 18
				},
				{
					id: 'osm-layer',
					type: 'raster',
					source: 'osm',
					layout: {
						visibility: currentBasemap === 'osm' ? 'visible' : 'none'
					},
					minzoom: 0,
					maxzoom: 19
				},
				{
					id: 'konjaku-layer',
					type: 'raster',
					source: 'konjaku',
					minzoom: 8,  // ここを8に変更！
					maxzoom: 16, // ここを16に変更！
					paint: {
						'raster-opacity': opacity
					}
				}
			]
		};
		return gsiStyle;
	}

	function showCurrentPosition() {
		if ('geolocation' in navigator) {
			navigator.geolocation.getCurrentPosition((position) => {
				const { latitude, longitude } = position.coords;
				
				if (map && !currentMarker) {
					currentMarker = new maplibregl.Marker()
						.setLngLat([longitude, latitude])
						.addTo(map);
					
					map.flyTo({
						center: [longitude, latitude],
						zoom: 15
					});
				}
			});
		}
	}

	function watchCurrentPosition() {
		if ('geolocation' in navigator) {
			navigator.geolocation.watchPosition((position) => {
				const { latitude, longitude } = position.coords;
				// 現在位置を更新
				currentPosition = { lat: latitude, lng: longitude };
				
				if (map) {
					if (!currentMarker) {
						// ここを変更！
						currentMarker = createLocationMarker(map)
							.setLngLat([longitude, latitude])
							.addTo(map);
						
						map.flyTo({
							center: [longitude, latitude],
							zoom: 15
						});
					} else {
						currentMarker.setLngLat([longitude, latitude]);
					}
				}
			}, (error) => {
				console.error('位置情報の取得に失敗したよ！', error);
			});
		}
	}

	// マーカーのスタイルを定義する関数を追加
	function createLocationMarker(map: maplibregl.Map) {
		// 外側の青い円
		const outer = document.createElement('div');
		outer.className = 'location-marker-outer';
		outer.style.width = '30px';
		outer.style.height = '30px';
		outer.style.borderRadius = '50%';
		outer.style.backgroundColor = 'rgba(37, 99, 235, 0.2)';
		outer.style.border = '2px solid white';

		// 内側の青い円
		const inner = document.createElement('div');
		inner.className = 'location-marker-inner';
		inner.style.width = '20px';
		inner.style.height = '20px';
		inner.style.borderRadius = '50%';
		inner.style.backgroundColor = 'rgb(37, 99, 235)';
		inner.style.border = '0.5px solid white';
		inner.style.position = 'absolute';
		inner.style.top = '50%';
		inner.style.left = '50%';
		inner.style.transform = 'translate(-50%, -50%)';

		outer.appendChild(inner);

		return new maplibregl.Marker({
			element: outer,
			anchor: 'center'
		});
	}

	// 透明度が変更されたときにマップのレイヤーを更新
	$effect(() => {
		if (map) {
			map.setPaintProperty('konjaku-layer', 'raster-opacity', opacity);
		}
	});

	onMount(() => {
		watchCurrentPosition();
	});

	interface Memo {
		id: string;
		lat: number;
		lng: number;
		text: string;
		date: string;
		marker?: maplibregl.Marker;
	}

	let memos: Memo[] = $state([]);
	let currentMemo = $state('');
	let editingMemo: Memo | null = $state(null);  // 編集中のメモ

	async function addMemo(e?: maplibregl.MapMouseEvent) {
		const position = e 
			? { lat: e.lngLat.lat, lng: e.lngLat.lng }
			: currentPosition;

		if (!position) {
			console.error('位置情報が取得できてないよ！');
			return;
		}

		const date = new Date().toISOString();
		const id = crypto.randomUUID();

		// マーカーを作成
		const marker = new maplibregl.Marker()
			.setLngLat([position.lng, position.lat])
			.addTo(map!);

		const newMemo: Memo = {
			id,
			lat: position.lat,
			lng: position.lng,
			text: currentMemo,
			date,
			marker
		};

		memos = [...memos, newMemo];
		currentMemo = '';
	}

	function startEdit(memo: Memo) {
		editingMemo = memo;
		currentMemo = memo.text;
	}

	function updateMemo() {
		if (!editingMemo) return;

		const currentEditingMemo = editingMemo; // ローカル変数に代入
		
		memos = memos.map(m => 
			m.id === currentEditingMemo.id 
				? { ...m, text: currentMemo }
				: m
		);
		
		editingMemo = null;
		currentMemo = '';
	}

	function deleteMemo(id: string) {
		const memo = memos.find(m => m.id === id);
		if (memo?.marker) memo.marker.remove(); // マーカーを削除
		memos = memos.filter(m => m.id !== id);
	}

	function exportCSV() {
		const headers = ['緯度', '経度', 'メモ', '日時'];
		const rows = memos.map(m => [m.lat, m.lng, m.text, m.date]);
		const csvContent = [
			headers.join(','),
			...rows.map(row => row.join(','))
		].join('\n');

		const blob = new Blob([csvContent], { type: 'text/csv' });
		const url = URL.createObjectURL(blob);
		const a = document.createElement('a');
		a.href = url;
		a.download = `memos_${new Date().toISOString().split('T')[0]}.csv`;
		document.body.appendChild(a);
		a.click();
		document.body.removeChild(a);
		URL.revokeObjectURL(url);
	}

	let pressTimer: number | null = null;
	let isPressing = false;

	function handleTouchStart(e: maplibregl.MapTouchEvent) {
		isPressing = true;
		pressTimer = window.setTimeout(() => {
			if (isPressing && map) {
				const touch = e.originalEvent.touches[0];  // originalEventを使う！
				const point = map.project([touch.clientX, touch.clientY]);
				const lngLat = map.unproject(point);
				
				addMemo({
					lngLat,
					preventDefault: () => e.preventDefault()
				} as maplibregl.MapMouseEvent);
			}
		}, 800); // 0.8秒の長押しで発動
	}

	function handleTouchEnd(e: maplibregl.MapTouchEvent) {
		isPressing = false;
		if (pressTimer) {
			clearTimeout(pressTimer);
			pressTimer = null;
		}
	}

	function handleTouchMove(e: maplibregl.MapTouchEvent) {
		isPressing = false;
		if (pressTimer) {
			clearTimeout(pressTimer);
			pressTimer = null;
		}
	}

    // タブの型定義
    type TabType = 'opacity' | 'memo' | 'list';
    
    // アクティブなタブの状態管理
    let activeTab: TabType = $state('opacity');  // デフォルトはopacityタブ
</script>

<div class="h-full w-screen">
    <!-- タイトルヘッダー追加！ -->
    <div class="absolute top-0 left-0 right-0 bg-white/90 p-2 text-center shadow-sm z-10">
        <h1 class="text-lg font-bold text-gray-700">古地図さんぽ</h1>
    </div>

    <MapLibre
        bind:map
        class="h-[100vh]"
        style={updateStyle()} 
        zoom={15}
        center={{ lng: 135.80578, lat: 34.89518 }}
        maxZoom={17.9}
        ondblclick={addMemo}
        ontouchstart={handleTouchStart}
        ontouchend={handleTouchEnd}
        ontouchmove={handleTouchMove}
    />

    <!-- 左上のベースマップ切り替えを少し下にずらす -->
    <div class="absolute top-14 left-4 bg-white p-2 rounded-lg shadow-lg z-10">
        <select
            bind:value={currentBasemap}
            class="text-sm p-1.5 border rounded"
            onchange={() => {
                if (map) map.setStyle(updateStyle());
            }}
        >
            {#each Object.entries(basemaps) as [value, label]}
                <option {value}>{label}</option>
            {/each}
        </select>
    </div>

    <!-- 右上の出展表示も少し下にずらす -->
    <div class="absolute top-14 right-4 bg-white/80 p-2 rounded-lg shadow-lg z-10 text-xs text-gray-600">
        <div>地図データ：</div>
        <div> 国土地理院</div>
        <div>OpenStreetMap contributors</div>
        <div> 今昔マップ on the web</div>
		<div>（京阪神圏1922-1923）</div>
    </div>

    <!-- 下部のコントロールパネルはそのまま -->
    <div class="absolute bottom-0 left-0 right-0 bg-white shadow-lg z-10">
        <div class="flex border-b">
            <button
                class="flex-1 p-2 text-sm {activeTab === 'opacity' ? 'bg-blue-50 border-b-2 border-blue-500' : ''}"
                onclick={() => activeTab = 'opacity'}
            >
                透明度
            </button>
            <button
                class="flex-1 p-2 text-sm {activeTab === 'memo' ? 'bg-blue-50 border-b-2 border-blue-500' : ''}"
                onclick={() => activeTab = 'memo'}
            >
                メモ
            </button>
            <button
                class="flex-1 p-2 text-sm {activeTab === 'list' ? 'bg-blue-50 border-b-2 border-blue-500' : ''}"
                onclick={() => activeTab = 'list'}
            >
                リスト({memos.length})
            </button>
        </div>

        <div class="p-3">
            {#if activeTab === 'opacity'}
                <label class="block">
                    <span class="text-sm">今昔マップの不透明度</span>
                    <input
                        type="range"
                        min="0"
                        max="1"
                        step="0.1"
                        bind:value={opacity}
                        class="w-full"
                    />
                </label>
            {:else if activeTab === 'memo'}
                <div class="flex flex-col gap-2">
                    <input
                        type="text"
                        bind:value={currentMemo}
                        placeholder={editingMemo ? "メモを編集" : "メモを入力してください"}
                        class="w-full p-2 text-sm border rounded"
                    />
                    {#if editingMemo}
                        <div class="flex gap-2">
                            <button
                                onclick={updateMemo}
                                class="flex-1 bg-green-500 text-white p-2 text-sm rounded"
                            >
                                更新
                            </button>
                            <button
                                onclick={() => {editingMemo = null; currentMemo = '';}}
                                class="flex-1 bg-gray-500 text-white p-2 text-sm rounded"
                            >
                                キャンセル
                            </button>
                        </div>
                    {:else}
                        <button
                            onclick={() => addMemo()}
                            class="w-full bg-blue-500 text-white p-2 text-sm rounded"
                            disabled={!currentPosition}
                        >
                            現在地にメモを追加
                        </button>
                    {/if}
                </div>
            {:else if activeTab === 'list'}
                <div class="flex flex-col gap-2">
                    {#if memos.length > 0}
                        <div class="max-h-48 overflow-y-auto">
                            {#each memos as memo}
                                <div class="text-sm border-b p-2">
                                    <div class="flex justify-between">
                                        <div>{memo.text}</div>
                                        <div class="flex gap-2">
                                            <button
                                                onclick={() => startEdit(memo)}
                                                class="text-blue-500"
                                            >
                                                ✏️
                                            </button>
                                            <button
                                                onclick={() => deleteMemo(memo.id)}
                                                class="text-red-500"
                                            >
                                                🗑️
                                            </button>
                                        </div>
                                    </div>
                                    <div class="text-xs text-gray-500">
                                        ({memo.lat.toFixed(6)}, {memo.lng.toFixed(6)})
                                    </div>
                                </div>
                            {/each}
                        </div>
                        <!-- CSV出力ボタンを追加 -->
                        <button
                            onclick={exportCSV}
                            class="w-full bg-green-500 text-white p-2 text-sm rounded mt-2 flex items-center justify-center gap-2"
                        >
                            <span>📥</span>
                            <span>CSVファイルに出力</span>
                        </button>
                    {:else}
                        <div class="text-sm text-gray-500 text-center p-4">
                            メモがまだないよ！地図をダブルクリックしてメモを追加してね
                        </div>
                    {/if}
                </div>
            {/if}
        </div>
    </div>
</div>
