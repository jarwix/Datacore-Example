```datacorejsx
return function View() {
/*
 * © 2025 Jarwix
 *
 * Этот шаблон предоставляется по лицензии MIT.
 * Вы можете использовать, изменять и распространять его свободно,
 * при условии обязательного указания автора и включения этой лицензии.
 *
 * Author: Jarwix (https://t.me/sdvghack)
 */

  const currentFile = dc.currentFile();

  const getFilePath = () => {
    return currentFile.value("$path") || currentFile.path || currentFile.file?.path;
  };

  const getFmValue = (key, def) => {

    const val = currentFile.value(key);
    return val !== undefined && val !== null ? Number(val) : def;
  };

  const getFmUpgrades = () => {
    const val = currentFile.value("clicker_upgrades");
    return val ? val : {};
  };

  const [game, setGame] = dc.useState(() => {
    const path = getFilePath();
    console.log("DEBUG: Init path:", path);
    return {
      score: getFmValue("clicker_score", 0),
      autoPerSec: getFmValue("clicker_auto", 0),
      clickPower: getFmValue("clicker_power", 1),
      upgrades: getFmUpgrades()
    };
  });

  const gameRef = dc.useRef(game);

  dc.useEffect(() => {
    gameRef.current = game;
  }, [game]);

  const saveToFrontmatter = async () => {

    const data = gameRef.current;

    const app = window.app;
    if (!app) return;

    const path = getFilePath();
    if (!path) {
      console.error("CRITICAL: No path");
      return;
    }

    const tFile = app.vault.getAbstractFileByPath(path);

    if (tFile) {
      try {
        console.log(`DEBUG: Saving score ${data.score.toFixed(2)} to FM...`); 
        await app.fileManager.processFrontMatter(tFile, (fm) => {

          fm.clicker_score = data.score; 
          fm.clicker_auto = data.autoPerSec;
          fm.clicker_power = data.clickPower;
          fm.clicker_upgrades = data.upgrades;
        });

      } catch (e) {
        console.error("Save Error:", e);
      }
    }
  };

  dc.useEffect(() => {
    const interval = setInterval(() => {

      if (gameRef.current.score > 0 || gameRef.current.autoPerSec > 0) {
          saveToFrontmatter();
      }
    }, 5000);
    return () => clearInterval(interval);
  }, []); 

  dc.useEffect(() => {
    if (game.autoPerSec > 0) {
      const interval = setInterval(() => {

        setGame(prev => ({ ...prev, score: prev.score + prev.autoPerSec }));
      }, 1000);
      return () => clearInterval(interval);
    }
  }, [game.autoPerSec]);

  const click = () => {
    setGame(prev => ({ ...prev, score: prev.score + prev.clickPower }));
  };

  const upgradesList = [
    { id: "doubleClick", name: "Двойной клик", baseCost: 50, effect: (g) => ({ clickPower: g.clickPower + 1 }) },
    { id: "auto1", name: "Робот-помощник", baseCost: 100, effect: (g) => ({ autoPerSec: g.autoPerSec + 1 }) },
    { id: "megaClick", name: "Мега-клик ×5", baseCost: 500, effect: (g) => ({ clickPower: g.clickPower + 5 }) },
    { id: "factory", name: "Фабрика печенья", baseCost: 2000, effect: (g) => ({ autoPerSec: g.autoPerSec + 10 }) },
  ];

  const buyUpgrade = (upg) => {
    const owned = game.upgrades[upg.id] || 0;
    const cost = Math.floor(upg.baseCost * Math.pow(1.15, owned));

    if (game.score >= cost) {
      const effectResult = upg.effect(game);
      setGame(prev => ({
        ...prev,
        score: prev.score - cost,
        upgrades: { ...prev.upgrades, [upg.id]: owned + 1 },
        ...effectResult 
      }));
    }
  };

  const resetGame = async () => {
    const cleanState = { score: 0, autoPerSec: 0, clickPower: 1, upgrades: {} };
    setGame(cleanState);

    const app = window.app;
    const path = getFilePath();
    if (path) {
        const tFile = app.vault.getAbstractFileByPath(path);
        if (tFile) {
            await app.fileManager.processFrontMatter(tFile, (fm) => {
                delete fm.clicker_score;
                delete fm.clicker_auto;
                delete fm.clicker_power;
                delete fm.clicker_upgrades;
            });
        }
    }
  };

  return (
    <div style={{ 
      fontFamily: "var(--font-interface)", 
      textAlign: "center", 
      padding: "20px", 
      maxWidth: "500px", 
      margin: "0 auto",
      border: "1px solid var(--background-modifier-border)",
      borderRadius: "10px",
      background: "var(--background-secondary)"
    }}>
      <h3>🍪 Кликер</h3>

      <div style={{ fontSize: "2.5em", margin: "20px 0", fontWeight: "bold", color: "var(--text-accent)" }}>
        {}
        {Math.floor(game.score)}
      </div>

      <div style={{ display: "flex", justifyContent: "center", gap: "20px", marginBottom: "20px", fontSize: "0.9em", color: "var(--text-muted)" }}>
         <span>👆 Сила: {game.clickPower}</span>
         <span>⚡ Авто: {game.autoPerSec}/сек</span>
      </div>

      <button 
        onClick={click}
        style={{
          fontSize: "40px", 
          width: "120px", 
          height: "120px", 
          borderRadius: "50%",
          background: "var(--interactive-accent)", 
          border: "none", 
          cursor: "pointer",
          boxShadow: "0 4px 15px rgba(0,0,0,0.2)",
          transition: "transform 0.1s"
        }}
        onMouseDown={e => e.currentTarget.style.transform = "scale(0.95)"}
        onMouseUp={e => e.currentTarget.style.transform = "scale(1)"}
        onMouseLeave={e => e.currentTarget.style.transform = "scale(1)"}
      >
        🍪
      </button>

      <div style={{ display: "grid", gridTemplateColumns: "1fr 1fr", gap: "10px", margin: "30px 0" }}>
        {upgradesList.map(upg => {
            const owned = game.upgrades[upg.id] || 0;
            const cost = Math.floor(upg.baseCost * Math.pow(1.15, owned));
            const canBuy = game.score >= cost;

            return (
                <button 
                    key={upg.id}
                    onClick={() => buyUpgrade(upg)}
                    disabled={!canBuy}
                    style={{
                        padding: "10px",
                        background: canBuy ? "var(--interactive-normal)" : "var(--background-modifier-form-field)",
                        opacity: canBuy ? 1 : 0.5,
                        border: "1px solid var(--background-modifier-border)",
                        borderRadius: "8px",
                        cursor: canBuy ? "pointer" : "default",
                        display: "flex",
                        flexDirection: "column",
                        alignItems: "center"
                    }}
                >
                    <span style={{fontWeight: "bold"}}>{upg.name}</span>
                    <span style={{fontSize: "0.8em"}}>Ур. {owned} | 💰 {cost}</span>
                </button>
            )
        })}
      </div>

      <div style={{borderTop: "1px solid var(--background-modifier-border)", paddingTop: "15px"}}>
        <button 
            onClick={resetGame}
            style={{
                fontSize: "12px",
                background: "var(--background-modifier-error)",
                color: "white",
                border: "none",
                padding: "5px 10px",
                borderRadius: "4px",
                cursor: "pointer"
            }}
        >
            Сброс прогресса
        </button>
        <div style={{fontSize: "10px", color: "var(--text-muted)", marginTop: "5px"}}>
            Файл: {getFilePath() || "..."}
        </div>
      </div>
    </div>
  );
};
```
