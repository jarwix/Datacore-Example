---
Folder: Телефон
FrontColumns:
  - Процессор
  - Дисплей
NoteColumns:
  - Фото
  - Оценка
Level: 3
Font: 12
Lines: 0
Sorting: true
---
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
    const frontmatter = dc.currentFile();
    const folderPath = frontmatter.value("Folder");
    const levelText = frontmatter.value("Level");
    const fontSize = frontmatter.value("Font");

    const getArrayFromValue = (val) => {
        if (Array.isArray(val)) return val.map(s => s.trim());
        if (typeof val === 'string') return val.split(',').map(s => s.trim()).filter(Boolean);
        return [];
    };

    const columnNames = getArrayFromValue(frontmatter.value("NoteColumns"));
    const frontmatterFields = getArrayFromValue(frontmatter.value("FrontColumns"));
    const rowsPerPage = parseInt(frontmatter.value("Lines")) || 0;
    const sortAsc = frontmatter.value("Sorting") === true;

    if (!folderPath || (columnNames.length === 0 && frontmatterFields.length === 0)) {
        return <div>Укажите Папку и (Столбцы или FrontСтолбцы) в Frontmatter.</div>;
    }

    const revision = dc.useIndexUpdates();
    const allData = dc.useMemo(() => {
        let fileQ = `path("${folderPath}")`;
        const files = dc.query(fileQ);

        let sections = [];
        if (columnNames.length > 0) {
            let secQ = `path("${folderPath}") and @section`;
            if (levelText) {
                secQ += ` and $level = ${levelText}`;
            }
            sections = dc.query(secQ);
        }

        return { files, sections };
    }, [revision]);

    const [rows, setRows] = dc.useState([]);
    const [loading, setLoading] = dc.useState(true);

    dc.useEffect(() => {
        const loadData = async () => {
            const { files, sections } = allData;
            const filePaths = files.map(f => f.$path);

            const sectionsByFile = new Map();
            if (sections.length > 0) {
                sections.forEach(section => {
                    const filePath = section.$file;
                    if (!sectionsByFile.has(filePath)) {
                        sectionsByFile.set(filePath, {});
                    }
                    const fileData = sectionsByFile.get(filePath);
                    for (const colName of columnNames) {
                        if (section.$name.toLowerCase() === colName.toLowerCase() && !fileData[colName]) {
                            fileData[colName] = section;
                            break;
                        }
                    }
                });
            }

            const fileLinesCache = new Map();
            const imageCache = new Map();

            const uniqueFilePaths = filePaths.filter(filePath => {
                const hasSectionMatch = sectionsByFile.has(filePath) && Object.keys(sectionsByFile.get(filePath)).length > 0;
                if (hasSectionMatch) return true;

                const tfile = app.vault.getAbstractFileByPath(filePath);
                const cache = tfile ? app.metadataCache.getFileCache(tfile) : null;
                const fileFrontmatter = cache?.frontmatter || {};
                const hasFrontMatch = frontmatterFields.some(name => {
                    for (const [key] of Object.entries(fileFrontmatter)) {
                        if (key.toLowerCase() === name.toLowerCase()) return true;
                    }
                    return false;
                });
                return hasFrontMatch;
            }).sort((a, b) => {
                const comparison = a.localeCompare(b);
                return sortAsc ? comparison : -comparison;
            });

            if (columnNames.length > 0) {
                const filesNeedingLines = uniqueFilePaths.filter(p => sectionsByFile.has(p));
                await Promise.all(filesNeedingLines.map(async (filePath) => {
                    const tfile = app.vault.getAbstractFileByPath(filePath);
                    if (tfile) {
                        const content = await app.vault.read(tfile);
                        fileLinesCache.set(filePath, content.split('\n'));
                    }
                }));
            }

            const resolveImagePath = (embedSyntax) => {
                if (imageCache.has(embedSyntax)) return imageCache.get(embedSyntax);

                const match = embedSyntax.match(/!\[\[([^\]\|]+)(?:\|[^\]]*)?\]\]/);
                if (!match) return null;
                const rawPath = match[1].trim();

                const IMG_EXTS = ["jpg", "jpeg", "png", "gif", "webp", "svg"];
                const normalize = (p) => p.replace(/^\//, "").replace(/\/$/, "");
                const q = normalize(rawPath);
                if (!q) return null;

                const hasExt = /\.[a-z0-9]+$/i.test(q);
                const qBase = hasExt ? q.replace(/\.[^/.]+$/, "") : q;
                const qName = qBase.split("/").pop();
                const dirFromQ = qBase.includes("/") ? qBase.split("/").slice(0, -1).join("/") : "";
                const dirs = [dirFromQ].filter(Boolean).map(normalize);
                if (!dirs.length) dirs.push("");

                const exts = hasExt ? [q.split(".").pop().toLowerCase()] : IMG_EXTS;

                let f = app.vault.getAbstractFileByPath(q);
                if (f) {
                    const src = app.vault.getResourcePath(f);
                    imageCache.set(embedSyntax, src);
                    return src;
                }

                for (const dir of dirs) {
                    for (const ext of exts) {
                        const p = dir ? `${dir}/${qName}.${ext}` : `${qName}.${ext}`;
                        f = app.vault.getAbstractFileByPath(p);
                        if (f) {
                            const src = app.vault.getResourcePath(f);
                            imageCache.set(embedSyntax, src);
                            return src;
                        }
                    }
                }

                const filesList = app.vault.getFiles();
                const candidate = filesList.find(f =>
                    IMG_EXTS.some(ext => f.name.toLowerCase() === `${qName}.${ext}`.toLowerCase())
                );
                if (candidate) {
                    const src = app.vault.getResourcePath(candidate);
                    imageCache.set(embedSyntax, src);
                    return src;
                }

                imageCache.set(embedSyntax, null);
                return null;
            };

            const loadedRows = uniqueFilePaths.map(filePath => {
                const tfile = app.vault.getAbstractFileByPath(filePath);
                const cache = tfile ? app.metadataCache.getFileCache(tfile) : null;
                const fileFrontmatter = cache?.frontmatter || {};
                const data = sectionsByFile.get(filePath) || {};
                const row = { file: dc.fileLink(filePath) };
                const lines = columnNames.length > 0 && sectionsByFile.has(filePath) ? (fileLinesCache.get(filePath) || []) : [];
                columnNames.forEach(name => {
                    row[name] = extractContent(data[name], lines, fontSize, resolveImagePath);
                });
                frontmatterFields.forEach(name => {
				    let foundValue = '—';
				    for (const [key, value] of Object.entries(fileFrontmatter)) {
				        if (key.toLowerCase() === name.toLowerCase()) {
				            foundValue = value ?? '—';
				            break;
				        }
				    }
				    row[name] = foundValue === '—' 
				        ? '—' 
				        : <div style={{
				              fontSize: `${fontSize}px`,
				              lineHeight: 1.4,
				              display: 'flex',
				              flexDirection: 'column',
				              gap: '4px'
				          }}>
				              {typeof foundValue === 'string' 
				                  ? foundValue.split('\n').map((line, i) => 
				                      <span key={i}>{line || <br/>}</span>
				                  )
				                  : foundValue}
				          </div>;
				});
                return row;
            });

            setRows(loadedRows);
            setLoading(false);
        };

        loadData();
    }, [revision]);

    const extractContent = (section, lines, fontSize, resolveImagePath) => {
        if (!section || lines.length === 0) return '—';

        const elements = [];

        section.$blocks.forEach(block => {
            if (block.$type === 'list') {
                block.$elements.forEach(item => {
                    elements.push(<div key={elements.length}>• {item.$text}</div>);
                });
            }
            else if (block.$type === 'task') {
                block.$elements.forEach(item => {
                    elements.push(<div key={elements.length}>- [ ] {item.$text}</div>);
                });
            }
            else if (block.$type === 'paragraph') {
                const start = block.$position.start;
                const end = block.$position.end;
                const rawText = lines.slice(start, end).join('\n');

                const parts = rawText.split(/(!\[\[[^\]]+\]\])/g);

                parts.forEach(part => {
                    if (!part) return;

                    if (part.startsWith('![[')) {
                        const src = resolveImagePath(part);
                        if (src) {
                            elements.push(
                                <img
                                    key={elements.length}
                                    src={src}
                                    alt={part.match(/!\[\[([^\]\|]+)/)?.[1] || 'image'}
                                    style={{
                                        maxWidth: '100%',
                                        maxHeight: '180px',
                                        objectFit: 'contain',
                                        borderRadius: '6px',
                                        margin: '6px 0',
                                        display: 'block'
                                    }}
                                />
                            );
                        } else {
                            elements.push(<span key={elements.length}>[Фото не найдено]</span>);
                        }
                    } else {
                        const text = part.trim();
                        if (text) elements.push(<span key={elements.length}>{text}</span>);
                    }
                });
            }
        });
        if (elements.length === 0) return '—';
        return (
            <div style={{
                fontSize: `${fontSize}px`,
                lineHeight: 1.4,
                display: 'flex',
                flexDirection: 'column',
                gap: '4px'
            }}>
                {elements}
            </div>
        );
    };

    if (loading) {
        return <div>Загрузка данных...</div>;
    }

    const COLUMNS = [
        { id: "Файл", value: row => row.file },
        ...frontmatterFields.map(name => ({
            id: name,
            value: row => row[name]
        })),
        ...columnNames.map(name => ({
            id: name,
            value: row => row[name]
        }))
    ];

    return <dc.Table {...(rowsPerPage > 0 ? { paging: rowsPerPage } : {})} columns={COLUMNS} rows={rows} />;
};
```
