import { useCallback, useEffect, useMemo, useRef, useState } from "react";
import {
  Thermometer,
  Droplets,
  CloudRain,
  Wind,
  MapPin,
  RefreshCw,
  Languages,
  Loader2,
  AlertTriangle,
  Sprout,
  Map as MapIcon,
  Trash2,
  Bookmark,
} from "lucide-react";
import FarmMap, { type LatLng } from "./components/FarmMap";
import AlertBanner from "./components/AlertBanner";
import SoilMeter from "./components/SoilMeter";
import CropCards from "./components/CropCards";
import { supabase } from "./lib/supabase";
import {
  b,
  districts,
  t,
  ui,
  type Lang,
  type Season,
} from "./i18n";
import {
  alertType,
  fetchWeather,
  formatHour,
  recommendCrops,
  type WeatherData,
} from "./weather";

type SeasonOption = { id: Season; key: keyof typeof ui };

const seasonOptions: Record<string, SeasonOption[]> = {
  sl: [
    { id: "maha", key: "maha" },
    { id: "yala", key: "yala" },
  ],
  tn: [
    { id: "samba", key: "samba" },
    { id: "kuruvai", key: "kuruvai" },
  ],
};

type SavedPin = {
  id: string;
  name: string | null;
  lat: number;
  lng: number;
  region: string | null;
};

export default function App() {
  const [lang, setLang] = useState<Lang>("en");
  const [weather, setWeather] = useState<WeatherData | null>(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const [selectedDistrict, setSelectedDistrict] = useState<string>("sl-vavuniya");
  const [customLocation, setCustomLocation] = useState<LatLng | null>(null);
  const [locating, setLocating] = useState(false);
  const [districtSearch, setDistrictSearch] = useState("");

  const district = useMemo(
    () => districts.find((d) => d.id === selectedDistrict) ?? districts[0],
    [selectedDistrict]
  );
  const [season, setSeason] = useState<Season>("maha");

  const [pin, setPin] = useState<LatLng | null>(null);
  const [pinWeather, setPinWeather] = useState<WeatherData | null>(null);
  const [pinLoading, setPinLoading] = useState(false);

  const [savedPins, setSavedPins] = useState<SavedPin[]>([]);
  const mapCenter = pin ?? customLocation ?? { lat: district.lat, lng: district.lng };

  const loadWeather = useCallback(async (lat: number, lng: number) => {
    setLoading(true);
    setError(null);
    try {
      const data = await fetchWeather(lat, lng);
      setWeather(data);
    } catch (e) {
      setError(e instanceof Error ? e.message : "failed");
    } finally {
      setLoading(false);
    }
  }, []);

  useEffect(() => {
    loadWeather(district.lat, district.lng);
  }, [district.lat, district.lng, loadWeather]);

  const alert = weather ? alertType(weather) : "normal";
  const recs = useMemo(
    () => (weather ? recommendCrops(weather, season) : []),
    [weather, season]
  );

  // keep default season in sync with the chosen region
  useEffect(() => {
    setSeason(district.region === "sl" ? "maha" : "samba");
  }, [district.region]);

  // Saved pins
  const pinsLoaded = useRef(false);
  useEffect(() => {
    if (pinsLoaded.current) return;
    pinsLoaded.current = true;
    supabase
      .from("farm_locations")
      .select("*")
      .order("created_at", { ascending: false })
      .limit(10)
      .then(({ data }) => {
        if (data) setSavedPins(data as SavedPin[]);
      });
  }, []);

  async function savePin() {
    if (!pin) return;
    const { data, error: err } = await supabase
      .from("farm_locations")
      .insert({
        lat: pin.lat,
        lng: pin.lng,
        region: district.name.en,
        name: null,
      })
      .select()
      .single();
    if (!err && data) {
      setSavedPins((prev) => [data as SavedPin, ...prev].slice(0, 10));
      setPin(null);
    }
  }

  async function removePin(id: string) {
    setSavedPins((prev) => prev.filter((p) => p.id !== id));
    await supabase.from("farm_locations").delete().eq("id", id);
  }

  async function handlePinWeather() {
    if (!pin) return;
    setPinLoading(true);
    setPinWeather(null);
    try {
      const data = await fetchWeather(pin.lat, pin.lng);
      setPinWeather(data);
    } catch {
      setPinWeather(null);
    } finally {
      setPinLoading(false);
    }
  }

  function detectLocation() {
    if (!navigator.geolocation) return;
    setLocating(true);
    navigator.geolocation.getCurrentPosition(
      (pos) => {
        const loc = { lat: pos.coords.latitude, lng: pos.coords.longitude };
        setCustomLocation(loc);
        setPin(null);
        setLocating(false);
        loadWeather(loc.lat, loc.lng);
      },
      () => setLocating(false),
      { timeout: 10000 }
    );
  }

  function pickOnMap(p: LatLng) {
    setPin(p);
    setPinWeather(null);
  }

  const regionTabs: { id: "sl" | "tn"; label: keyof typeof ui }[] = [
    { id: "sl", label: "sriLanka" },
    { id: "tn", label: "tamilNadu" },
  ];
  const filteredDistricts = districts.filter((d) => d.region === district.region);

  return (
    <div className="min-h-screen bg-[#f5f3ee] text-stone-800">
      {/* Header */}
      <header className="bg-gradient-to-r from-green-800 to-green-700 text-white shadow-md">
        <div className="max-w-6xl mx-auto px-4 py-4 flex items-center justify-between gap-3">
          <div className="flex items-center gap-3 min-w-0">
            <div className="rounded-xl bg-white/15 p-2 shrink-0">
              <Sprout className="h-7 w-7" />
            </div>
            <div className="min-w-0">
              <h1 className="text-lg sm:text-xl font-bold truncate">{t("appTitle", lang)}</h1>
              <p className="text-xs sm:text-sm text-green-100 truncate">{t("appSubtitle", lang)}</p>
            </div>
          </div>
          <button
            onClick={() => setLang((l) => (l === "en" ? "ta" : "en"))}
            className="shrink-0 inline-flex items-center gap-2 rounded-full bg-white text-green-800 font-bold px-4 py-2.5 hover:bg-green-50 active:scale-95 transition"
          >
            <Languages className="h-5 w-5" />
            {t("languageName", lang)}
          </button>
        </div>
      </header>

      <main className="max-w-6xl mx-auto px-4 py-6 space-y-6">
        {/* Location picker */}
        <section className="rounded-2xl bg-white border border-stone-200 shadow-sm p-5">
          <div className="flex flex-col sm:flex-row sm:items-end gap-4">
            <div className="flex-1 grid gap-3 sm:grid-cols-2">
              <div>
                <label className="text-xs font-bold uppercase tracking-wide text-stone-500 block mb-1.5">
                  {t("regionLabel", lang)}
                </label>
                <div className="flex rounded-xl bg-stone-100 p-1">
                  {regionTabs.map((r) => (
                    <button
                      key={r.id}
                      onClick={() => {
                        const first = districts.find((d) => d.region === r.id);
                        if (first) setSelectedDistrict(first.id);
                      }}
                      className={`flex-1 rounded-lg px-3 py-2 text-sm font-semibold transition ${
                        district.region === r.id
                          ? "bg-green-700 text-white shadow"
                          : "text-stone-600 hover:text-stone-800"
                      }`}
                    >
                      {t(r.label, lang)}
                    </button>
                  ))}
                </div>
              </div>
              <div>
                <label className="text-xs font-bold uppercase tracking-wide text-stone-500 block mb-1.5">
                  {t("selectDistrict", lang)}
                </label>
                <input
                  value={districtSearch}
                  onChange={(e) => setDistrictSearch(e.target.value)}
                  placeholder={t("searchDistrict", lang)}
                  className="w-full rounded-xl border border-stone-300 bg-white px-3 py-2.5 text-sm font-medium focus:outline-none focus:ring-2 focus:ring-green-600 mb-2"
                />
                <select
                  value={selectedDistrict}
                  onChange={(e) => {
                    setSelectedDistrict(e.target.value);
                    setCustomLocation(null);
                    setPin(null);
                    setPinWeather(null);
                  }}
                  className="w-full rounded-xl border border-stone-300 bg-white px-3 py-2.5 text-sm font-medium focus:outline-none focus:ring-2 focus:ring-green-600"
                >
                  {filteredDistricts
                    .filter((d) =>
                      districtSearch.trim() === ""
                        ? true
                        : d.name.en.toLowerCase().includes(districtSearch.trim().toLowerCase()) ||
                          d.name.ta.includes(districtSearch.trim())
                    )
                    .map((d) => (
                      <option key={d.id} value={d.id}>
                        {b(d.name, lang)}
                      </option>
                    ))}
                </select>
              </div>
            </div>
            <button
              onClick={detectLocation}
              disabled={locating}
              className="inline-flex items-center justify-center gap-2 rounded-xl bg-amber-500 hover:bg-amber-600 disabled:opacity-60 text-white font-bold px-4 py-2.5 text-sm transition active:scale-95"
            >
              {locating ? (
                <Loader2 className="h-4 w-4 animate-spin" />
              ) : (
                <MapPin className="h-4 w-4" />
              )}
              {locating ? t("locating", lang) : t("useMyLocation", lang)}
            </button>
          </div>
          {customLocation && (
            <p className="mt-3 text-sm font-medium text-green-700 flex items-center gap-1.5">
              <MapPin className="h-4 w-4" /> {t("locationDetected", lang)} —{" "}
              {customLocation.lat.toFixed(3)}, {customLocation.lng.toFixed(3)}
            </p>
          )}
        </section>

        {/* Alert banner */}
        {weather && !error && (
          <>
            <AlertBanner alert={alert} lang={lang} />
            {weather.rainStart && (
              <p className="rounded-xl bg-blue-50 border border-blue-200 text-blue-800 px-4 py-3 text-sm font-semibold flex items-center gap-2">
                <CloudRain className="h-4 w-4 shrink-0" />
                {t("rainTiming", lang)}: {formatHour(weather.rainStart, lang)}
              </p>
            )}
          </>
        )}

        {/* Loading / error states */}
        {loading && (
          <div className="rounded-2xl bg-white border border-stone-200 p-10 flex flex-col items-center gap-3 text-stone-500">
            <Loader2 className="h-8 w-8 animate-spin text-green-700" />
            <p className="text-sm font-medium">{t("loading", lang)}</p>
          </div>
        )}
        {error && !loading && (
          <div className="rounded-2xl bg-red-50 border border-red-200 p-8 flex flex-col items-center gap-3 text-red-700">
            <AlertTriangle className="h-8 w-8" />
            <p className="text-sm font-medium text-center">{t("loadError", lang)}</p>
            <button
              onClick={() => loadWeather(mapCenter.lat, mapCenter.lng)}
              className="rounded-xl bg-red-600 text-white font-bold px-4 py-2 text-sm hover:bg-red-700 transition"
            >
              {t("retry", lang)}
            </button>
          </div>
        )}

        {/* Dashboard */}
        {weather && !loading && !error && (
          <>
            {/* Weather tiles */}
            <section className="grid grid-cols-2 lg:grid-cols-4 gap-4">
              {[
                {
                  icon: <Thermometer className="h-5 w-5" />,
                  label: t("temp", lang),
                  value: `${Math.round(weather.temperature)}°C`,
                  color: "bg-orange-100 text-orange-700",
                },
                {
                  icon: <Droplets className="h-5 w-5" />,
                  label: t("humidity", lang),
                  value: `${weather.humidity}%`,
                  color: "bg-sky-100 text-sky-700",
                },
                {
                  icon: <CloudRain className="h-5 w-5" />,
                  label: t("rainChance", lang),
                  value: `${weather.rainProbability}%`,
                  color: "bg-blue-100 text-blue-700",
                },
                {
                  icon: <Wind className="h-5 w-5" />,
                  label: t("wind", lang),
                  value: `${weather.windSpeed.toFixed(0)} km/h`,
                  color: "bg-teal-100 text-teal-700",
                },
              ].map((tile) => (
                <div
                  key={tile.label}
                  className="rounded-2xl bg-white border border-stone-200 shadow-sm p-4 sm:p-5"
                >
                  <div className={`rounded-xl inline-flex p-2.5 ${tile.color}`}>{tile.icon}</div>
                  <p className="mt-3 text-xs font-semibold text-stone-500 uppercase tracking-wide">
                    {tile.label}
                  </p>
                  <p className="text-2xl font-bold text-stone-800 mt-0.5">{tile.value}</p>
                </div>
              ))}
            </section>

            {/* Soil meter + season / crops */}
            <section className="grid gap-6 lg:grid-cols-[1fr_1.4fr]">
              <div className="space-y-6">
                <SoilMeter moisture={weather.soilMoisture} lang={lang} />

                <div className="rounded-2xl bg-white border border-stone-200 shadow-sm p-5">
                  <p className="text-xs font-bold uppercase tracking-wide text-stone-500 mb-3">
                    {t("seasonLabel", lang)}
                  </p>
                  <div className="grid grid-cols-2 gap-2">
                    {seasonOptions[district.region].map((s) => (
                      <button
                        key={s.id}
                        onClick={() => setSeason(s.id)}
                        className={`rounded-xl px-3 py-2.5 text-sm font-semibold border transition ${
                          season === s.id
                            ? "bg-green-700 text-white border-green-700 shadow"
                            : "bg-white text-stone-700 border-stone-300 hover:border-green-600"
                        }`}
                      >
                        {t(s.key, lang)}
                      </button>
                    ))}
                  </div>
                  <button
                    onClick={() => loadWeather(mapCenter.lat, mapCenter.lng)}
                    className="mt-4 w-full inline-flex items-center justify-center gap-2 rounded-xl border border-green-700 text-green-800 font-bold px-4 py-2.5 text-sm hover:bg-green-50 transition"
                  >
                    <RefreshCw className="h-4 w-4" /> {t("refresh", lang)}
                  </button>
                </div>
              </div>

              <div>
                <div className="mb-4">
                  <h2 className="text-lg font-bold text-stone-800 flex items-center gap-2">
                    <Sprout className="h-5 w-5 text-green-700" />
                    {t("cropTitle", lang)}
                  </h2>
                  <p className="text-sm text-stone-500">{t("cropSubtitle", lang)}</p>
                </div>
                <CropCards recs={recs} lang={lang} />
              </div>
            </section>
          </>
        )}

        {/* Map */}
        <section className="rounded-2xl bg-white border border-stone-200 shadow-sm p-5">
          <div className="flex items-start justify-between gap-3 flex-wrap mb-3">
            <div>
              <h2 className="text-lg font-bold text-stone-800 flex items-center gap-2">
                <MapIcon className="h-5 w-5 text-green-700" />
                {t("mapTitle", lang)}
              </h2>
              <p className="text-sm text-stone-500">{t("mapHint", lang)}</p>
            </div>
            {pin && (
              <button
                onClick={handlePinWeather}
                disabled={pinLoading}
                className="inline-flex items-center gap-2 rounded-xl bg-green-700 hover:bg-green-800 disabled:opacity-60 text-white font-bold px-4 py-2.5 text-sm transition"
              >
                {pinLoading ? (
                  <Loader2 className="h-4 w-4 animate-spin" />
                ) : (
                  <CloudRain className="h-4 w-4" />
                )}
                {t("tapPoint", lang)}
              </button>
            )}
          </div>

          <FarmMap
            lang={lang}
            center={mapCenter}
            pin={pin}
            onPick={pickOnMap}
            onSelectDistrict={(id) => {
              setSelectedDistrict(id);
              setCustomLocation(null);
              setPin(null);
              setPinWeather(null);
            }}
          />

          {pin && (
            <div className="mt-4 rounded-xl bg-stone-50 border border-stone-200 p-4 space-y-3">
              <p className="text-sm font-semibold text-stone-700">
                {t("tapPoint", lang)}: {pin.lat.toFixed(4)}, {pin.lng.toFixed(4)}
              </p>
              {pinWeather && (
                <div className="grid grid-cols-2 sm:grid-cols-4 gap-3 text-center">
                  <div className="rounded-lg bg-white border border-stone-200 p-2.5">
                    <p className="text-xs text-stone-500">{t("temp", lang)}</p>
                    <p className="font-bold">{Math.round(pinWeather.temperature)}°C</p>
                  </div>
                  <div className="rounded-lg bg-white border border-stone-200 p-2.5">
                    <p className="text-xs text-stone-500">{t("rainChance", lang)}</p>
                    <p className="font-bold">{pinWeather.rainProbability}%</p>
                  </div>
                  <div className="rounded-lg bg-white border border-stone-200 p-2.5">
                    <p className="text-xs text-stone-500">{t("humidity", lang)}</p>
                    <p className="font-bold">{pinWeather.humidity}%</p>
                  </div>
                  <div className="rounded-lg bg-white border border-stone-200 p-2.5">
                    <p className="text-xs text-stone-500">{t("soilMeter", lang)}</p>
                    <p className="font-bold">
                      {pinWeather.soilMoisture < 0.2
                        ? t("soilLow", lang)
                        : pinWeather.soilMoisture <= 0.35
                        ? t("soilOptimal", lang)
                        : t("soilHigh", lang)}
                    </p>
                  </div>
                </div>
              )}
              <div className="flex flex-wrap gap-2">
                <button
                  onClick={savePin}
                  className="inline-flex items-center gap-2 rounded-xl bg-amber-500 hover:bg-amber-600 text-white font-bold px-4 py-2 text-sm transition"
                >
                  <Bookmark className="h-4 w-4" /> {t("savePin", lang)}
                </button>
                <button
                  onClick={() => {
                    setPin(null);
                    setPinWeather(null);
                  }}
                  className="rounded-xl border border-stone-300 text-stone-600 font-semibold px-4 py-2 text-sm hover:bg-stone-100 transition"
                >
                  ✕
                </button>
              </div>
            </div>
          )}

          {/* Saved pins */}
          <div className="mt-4">
            <p className="text-xs font-bold uppercase tracking-wide text-stone-500 mb-2">
              {t("savedPins", lang)} ({savedPins.length})
            </p>
            {savedPins.length === 0 ? (
              <p className="text-sm text-stone-400">{t("noPins", lang)}</p>
            ) : (
              <ul className="flex flex-wrap gap-2">
                {savedPins.map((p) => (
                  <li
                    key={p.id}
                    className="inline-flex items-center gap-2 rounded-full bg-stone-100 border border-stone-200 pl-3 pr-1.5 py-1.5 text-xs font-medium"
                  >
                    <button
                      onClick={() => {
                        const loc = { lat: p.lat, lng: p.lng };
                        setPin(loc);
                        loadWeather(loc.lat, loc.lng);
                      }}
                      className="text-stone-700 hover:text-green-800"
                    >
                      <MapPin className="h-3.5 w-3.5 inline mr-1" />
                      {p.lat.toFixed(2)}, {p.lng.toFixed(2)}
                      {p.region ? ` · ${p.region}` : ""}
                    </button>
                    <button
                      onClick={() => removePin(p.id)}
                      title={t("removePin", lang)}
                      className="rounded-full p-1 hover:bg-red-100 text-stone-400 hover:text-red-600 transition"
                    >
                      <Trash2 className="h-3.5 w-3.5" />
                    </button>
                  </li>
                ))}
              </ul>
            )}
          </div>
        </section>

        <footer className="text-center text-xs text-stone-400 pb-6">
          {b(ui.appSubtitle, lang)} · Open-Meteo
        </footer>
      </main>
    </div>
  );
}
