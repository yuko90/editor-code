[README.txt](https://github.com/user-attachments/files/20581688/README.txt)
[Link.exe]https://mega.nz/file/bhxxEayA#o1T4JJ...Uv3mb6pyKxXN2E



import re
import tkinter as tk
from tkinter import filedialog, messagebox, ttk, simpledialog
import configparser
import os

# Couleurs pour le mode sombre (Notebook)
DARK_BG    = "#2e2e2e"  # fond principal du Notebook
DARK_FRAME = "#3e3e3e"  # fond onglets, etc.
LIGHT_FG   = "#f0f0f0"  # texte clair pour le Notebook

# Couleurs pour les éditeurs intégrés (Ini, MobCreate, Map, Obelisk)
DIR_BG       = "#2E2E2E"  # fond pour les éditeurs intégrés
DIR_FG       = "#FFFFFF"  # texte clair pour les éditeurs intégrés
DIR_BTN_BG   = "#3C3F41"  # boutons pour les éditeurs intégrés
DIR_BTN_FG   = "#FFFFFF"  # texte des boutons pour les éditeurs intégrés
DIR_ENTRY_BG = "#3C3F41"  # fond des Entry pour les éditeurs intégrés
DIR_ENTRY_FG = "#FFFFFF"  # texte des Entry pour les éditeurs intégrés

# Dictionnaire de traductions pour les textes de l'UI, avec mandarin ("zh")
TRANSLATIONS = {
    "fr": {
        "EditItemCreate":    "Édition Item Création",
        "EditMobCreate":     "Édition Mob Création",
        "EditMapCreate":     "Édition Carte Création",
        "EditObelisk":       "Édition Obélisque",
        "EditChaoticSquare": "Édition Carré Chaotique",
        "Langue":            "Langue",
        "Français":          "Français",
        "Anglais":           "Anglais",
        "Mandarin":          "Mandarin",
    },
    "en": {
        "EditItemCreate":    "Edit Item Create",
        "EditMobCreate":     "Edit Mob Create",
        "EditMapCreate":     "Edit Map Create",
        "EditObelisk":       "Edit Obelisk",
        "EditChaoticSquare": "Edit Chaotic Square",
        "Langue":            "Language",
        "Français":          "French",
        "Anglais":           "English",
        "Mandarin":          "Mandarin",
    },
    "zh": {
        "EditItemCreate":    "编辑物品创建",
        "EditMobCreate":     "编辑怪物创建",
        "EditMapCreate":     "编辑地图创建",
        "EditObelisk":       "编辑方尖碑",
        "EditChaoticSquare": "编辑混乱广场",
        "Langue":            "语言",
        "Français":          "法语",
        "Anglais":           "英语",
        "Mandarin":          "中文",
    }
}

def apply_dark_theme(widget):
    """
    Applique le thème sombre récursivement sur un widget et ses enfants.
    """
    try:
        widget.configure(bg=DIR_BG, fg=DIR_FG)
    except tk.TclError:
        pass
    for child in widget.winfo_children():
        apply_dark_theme(child)

# ──────────────────────────────────────────────────────────────────────────────
# INI EDITOR (onglet "EditItemCreate")
# ──────────────────────────────────────────────────────────────────────────────

class IniEditorFrame(tk.Frame):
    """
    Frame qui contient l’éditeur de fichier .ini (équivalent de IniEditorApp
    mais intégré dans un frame existant).
    """
    def __init__(self, parent):
        super().__init__(parent, bg=DIR_BG)
        self.sections = {}             # {section_name: [(id, qty), ...]}
        self.current_section = None    # Nom de la section sélectionnée
        self.file_path = None          # Chemin du fichier ouvert
        self.ratings = {}              # {(section, index): note_text}
        self.editing_entry = None      # Pour édition en cours dans le Treeview

        # Barre d'état
        self.status_var = tk.StringVar(value="Prêt")
        status_bar = tk.Label(
            self, textvariable=self.status_var, bd=1, relief=tk.SUNKEN,
            anchor=tk.W, bg=DIR_BTN_BG, fg=DIR_BTN_FG
        )
        status_bar.pack(side=tk.BOTTOM, fill=tk.X)

        # Conteneur principal
        main_frame = tk.Frame(self, bg=DIR_BG)
        main_frame.pack(fill=tk.BOTH, expand=True, padx=10, pady=10)

        # 1. Colonne Sections
        section_frame = tk.Frame(main_frame, bg=DIR_BG)
        section_frame.pack(side=tk.LEFT, fill=tk.Y, padx=(0,10))
        tk.Label(section_frame, text="Sections", bg=DIR_BG, fg=DIR_FG).pack()

        self.section_listbox = tk.Listbox(
            section_frame, width=25, bg=DIR_ENTRY_BG, fg=DIR_ENTRY_FG,
            selectbackground="#565A5C"
        )
        self.section_listbox.pack(fill=tk.Y, expand=True, pady=(5,0))
        self.section_listbox.bind("<<ListboxSelect>>", self.on_section_select)

        # Boutons Ajouter/Supprimer Section
        tk.Button(
            section_frame, text="➕ Ajouter C-Section", command=self.add_itemcreate_section,
            bg=DIR_BTN_BG, fg=DIR_BTN_FG
        ).pack(fill=tk.X, pady=(5,0))
        tk.Button(
            section_frame, text="➕ Ajouter R-Section", command=self.add_random_section,
            bg=DIR_BTN_BG, fg=DIR_BTN_FG
        ).pack(fill=tk.X, pady=2)
        tk.Button(
            section_frame, text="❌ Supprimer Section", command=self.remove_section,
            bg=DIR_BTN_BG, fg=DIR_BTN_FG
        ).pack(fill=tk.X, pady=2)

        # Boutons Charger/Sauvegarder en bas de la colonne Sections
        tk.Button(
            section_frame, text="ðﾟﾓﾂ Charger Fichier", command=self.open_file,
            bg=DIR_BTN_BG, fg=DIR_BTN_FG
        ).pack(fill=tk.X, pady=(20,2))
        tk.Button(
            section_frame, text="ðﾟﾒﾾ Sauvegarder Fichier", command=self.save_file,
            bg=DIR_BTN_BG, fg=DIR_BTN_FG
        ).pack(fill=tk.X, pady=2)

        # 2. Colonne Items (Treeview)
        item_frame = tk.Frame(main_frame, bg=DIR_BG)
        item_frame.pack(side=tk.LEFT, fill=tk.BOTH, expand=True)

        tk.Label(item_frame, text="Items", bg=DIR_BG, fg=DIR_FG).pack(anchor="nw")
        columns = ("ID", "Quantité", "Note", "Suppr")
        self.tree = ttk.Treeview(
            item_frame, columns=columns, show="headings", selectmode="browse"
        )
        self.tree.heading("ID", text="Grade")
        self.tree.heading("Quantité", text="% Rate")
        self.tree.heading("Note", text="Note")
        self.tree.heading("Suppr", text="")
        self.tree.column("ID", width=100, anchor=tk.CENTER)
        self.tree.column("Quantité", width=100, anchor=tk.CENTER)
        self.tree.column("Note", width=200, anchor=tk.W)
        self.tree.column("Suppr", width=50, anchor=tk.CENTER)
        self.tree.pack(fill=tk.BOTH, expand=True, pady=(5,0))

        # Lier clics sur le Treeview
        self.tree.bind("<Button-1>", self.on_tree_click)
        self.tree.bind("<Double-1>", self.on_double_click)

        # Recherche d'ID en haut du Treeview
        search_frame = tk.Frame(item_frame, bg=DIR_BG)
        search_frame.pack(fill=tk.X, pady=(5,0))
        self.search_var = tk.StringVar()
        tk.Entry(
            search_frame, textvariable=self.search_var, bg=DIR_ENTRY_BG,
            fg=DIR_ENTRY_FG, insertbackground=DIR_FG
        ).pack(side=tk.LEFT, fill=tk.X, expand=True)
        tk.Button(
            search_frame, text="ðﾟﾔﾍ", command=self.search_item,
            bg=DIR_BTN_BG, fg=DIR_BTN_FG
        ).pack(side=tk.RIGHT)

        # Bouton Nouvelle Ligne
        btn_frame = tk.Frame(item_frame, bg=DIR_BG)
        btn_frame.pack(fill=tk.X, pady=(5,0))
        tk.Button(
            btn_frame, text="➕ Nouvelle Ligne", command=self.add_blank_row,
            bg=DIR_BTN_BG, fg=DIR_BTN_FG
        ).pack(side=tk.LEFT, fill=tk.X, expand=True)

    # --- Fonctions Fichier ---
    def open_file(self):
        if self.sections:
            if not messagebox.askyesno("Confirmer", "Les modifications non enregistrées seront perdues. Continuer ?"):
                return
        path = filedialog.askopenfilename(
            filetypes=[("Text files", "*.txt;*.ini"), ("All files", "*.*")]
        )
        if not path:
            return
        self.file_path = path
        self.sections, self.ratings = self.parse_items_file(path)
        self.refresh_sections()
        self.clear_items()
        self.current_section = None
        self.status_var.set(f"Ouvert : {self.file_path}")

    def save_file(self):
        if not self.file_path:
            return self.save_file_as()
        try:
            self.write_items_file(self.file_path, self.sections, self.ratings)
            self.status_var.set(f"Enregistré : {self.file_path}")
        except Exception as e:
            messagebox.showerror("Erreur", f"Impossible de sauvegarder : {e}")

    def save_file_as(self):
        path = filedialog.asksaveasfilename(
            defaultextension=".ini", filetypes=[("INI files", "*.ini"), ("All files", "*.*")]
        )
        if not path:
            return
        self.file_path = path
        self.save_file()

    # --- Fonctions Sections ---
    def refresh_sections(self):
        def sort_key(name):
            m1 = re.match(r"^ItemCreate_(\d+)$", name)
            m2 = re.match(r"^RandomItemsCreate_(\d+)$", name)
            if m1:
                return (0, int(m1.group(1)))
            if m2:
                return (1, int(m2.group(1)))
            return (2, name.lower())
        sorted_names = sorted(self.sections.keys(), key=sort_key)

        self.section_listbox.delete(0, tk.END)
        for sec in sorted_names:
            self.section_listbox.insert(tk.END, sec)
        self.status_var.set(f"Sections: {len(self.sections)}")

    def on_section_select(self, event):
        sel = self.section_listbox.curselection()
        if not sel:
            return
        index = sel[0]
        section = self.section_listbox.get(index)
        self.current_section = section
        self.refresh_items()
        self.status_var.set(f"Section sélectionnée: {section}")

    def add_itemcreate_section(self):
        existing = [int(m.group(1)) for name in self.sections if (m := re.match(r"^ItemCreate_(\d+)$", name))]
        next_num = max(existing) + 1 if existing else 1
        new_name = f"ItemCreate_{next_num}"
        self.sections[new_name] = []
        self.refresh_sections()
        sorted_list = list(self.section_listbox.get(0, tk.END))
        idx = sorted_list.index(new_name)
        self.section_listbox.selection_clear(0, tk.END)
        self.section_listbox.selection_set(idx)
        self.section_listbox.see(idx)
        self.current_section = new_name
        self.refresh_items()
        self.status_var.set(f"Section ajoutée: {new_name}")

    def add_random_section(self):
        existing = [int(m.group(1)) for name in self.sections if (m := re.match(r"^RandomItemsCreate_(\d+)$", name))]
        next_num = max(existing) + 1 if existing else 1
        new_name = f"RandomItemsCreate_{next_num}"
        self.sections[new_name] = []
        self.refresh_sections()
        sorted_list = list(self.section_listbox.get(0, tk.END))
        idx = sorted_list.index(new_name)
        self.section_listbox.selection_clear(0, tk.END)
        self.section_listbox.selection_set(idx)
        self.section_listbox.see(idx)
        self.current_section = new_name
        self.refresh_items()
        self.status_var.set(f"Section ajoutée: {new_name}")

    def remove_section(self):
        if not self.current_section:
            return
        confirm = messagebox.askyesno("Confirmer", f"Supprimer la section [{self.current_section}] ?")
        if confirm:
            del self.sections[self.current_section]
            self.ratings = {k: v for k, v in self.ratings.items() if k[0] != self.current_section}
            self.current_section = None
            self.refresh_sections()
            self.clear_items()
            self.status_var.set("Section supprimée")

    # --- Lignes & Items ---
    def refresh_items(self):
        self.clear_items()
        if not self.current_section:
            return

        items = self.sections.get(self.current_section, [])
        total = len(items)
        for idx, (item_id, qty) in enumerate(items):
            note = self.ratings.get((self.current_section, idx), "")
            suppr_symbol = "❌" if total > 1 else ""
            self.tree.insert("", tk.END, iid=idx, values=(item_id, qty, note, suppr_symbol))

        self.status_var.set(f"Items dans '{self.current_section}': {total}")

    def clear_items(self):
        for row in self.tree.get_children():
            self.tree.delete(row)
        if self.editing_entry:
            self.editing_entry.destroy()
            self.editing_entry = None

    def add_blank_row(self):
        if not self.current_section:
            return
        self.sections[self.current_section].append((0, 0))
        self.refresh_items()
        new_idx = len(self.sections[self.current_section]) - 1
        self.tree.selection_set(new_idx)
        self.tree.see(new_idx)
        self.after(50, lambda: self.start_editing_cell(new_idx, 1))

    def start_editing_cell(self, row_idx, col_index):
        iid = str(row_idx)
        col = f"#{col_index}"
        bbox = self.tree.bbox(iid, col)
        if not bbox:
            return
        x, y, width, height = bbox
        if self.editing_entry:
            self.editing_entry.destroy()
        self.editing_entry = tk.Entry(self.tree, bg=DIR_ENTRY_BG, fg=DIR_ENTRY_FG, insertbackground=DIR_FG)
        self.editing_entry.place(x=x, y=y, width=width, height=height)
        if col_index == 1:
            current_val = str(self.sections[self.current_section][row_idx][0])
        elif col_index == 2:
            current_val = str(self.sections[self.current_section][row_idx][1])
        else:
            current_val = self.ratings.get((self.current_section, row_idx), "")
        self.editing_entry.insert(0, current_val)
        self.editing_entry.focus()

        def save_edit(event=None):
            if event.keysym == "Return" or event.type == tk.EventType.FocusOut:
                val = self.editing_entry.get()
                if col_index == 1:
                    try:
                        new_id = int(val)
                        qty = self.sections[self.current_section][row_idx][1]
                        self.sections[self.current_section][row_idx] = (new_id, qty)
                    except ValueError:
                        pass
                elif col_index == 2:
                    try:
                        new_qty = int(val)
                        item_id = self.sections[self.current_section][row_idx][0]
                        self.sections[self.current_section][row_idx] = (item_id, new_qty)
                    except ValueError:
                        pass
                else:
                    if val:
                        self.ratings[(self.current_section, row_idx)] = val
                    else:
                        self.ratings.pop((self.current_section, row_idx), None)
                self.editing_entry.destroy()
                self.editing_entry = None
                self.refresh_items()

        self.editing_entry.bind("<Return>", save_edit)
        self.editing_entry.bind("<FocusOut>", save_edit)

    def on_double_click(self, event):
        region = self.tree.identify("region", event.x, event.y)
        if region != "cell":
            return
        col = self.tree.identify_column(event.x)
        row = self.tree.identify_row(event.y)
        if not row:
            return
        idx = int(row)
        column_index = int(col.replace("#", ""))
        if column_index not in (1, 2, 3):
            return
        bbox = self.tree.bbox(row, col)
        if not bbox:
            return
        x, y, width, height = bbox
        if self.editing_entry:
            self.editing_entry.destroy()
        self.editing_entry = tk.Entry(self.tree, bg=DIR_ENTRY_BG, fg=DIR_ENTRY_FG, insertbackground=DIR_FG)
        self.editing_entry.place(x=x, y=y, width=width, height=height)
        if column_index == 1:
            current_val = str(self.sections[self.current_section][idx][0])
        elif column_index == 2:
            current_val = str(self.sections[self.current_section][idx][1])
        else:
            current_val = self.ratings.get((self.current_section, idx), "")
        self.editing_entry.insert(0, current_val)
        self.editing_entry.focus()

        def save_edit(event=None):
            if event.keysym == "Return" or event.type == tk.EventType.FocusOut:
                val = self.editing_entry.get()
                if column_index == 1:
                    try:
                        new_id = int(val)
                        qty = self.sections[self.current_section][idx][1]
                        self.sections[self.current_section][idx] = (new_id, qty)
                    except ValueError:
                        pass
                elif column_index == 2:
                    try:
                        new_qty = int(val)
                        item_id = self.sections[self.current_section][idx][0]
                        self.sections[self.current_section][idx] = (item_id, new_qty)
                    except ValueError:
                        pass
                else:
                    if val:
                        self.ratings[(self.current_section, idx)] = val
                    else:
                        self.ratings.pop((self.current_section, idx), None)
                self.editing_entry.destroy()
                self.editing_entry = None
                self.refresh_items()

        self.editing_entry.bind("<Return>", save_edit)
        self.editing_entry.bind("<FocusOut>", save_edit)

    def on_tree_click(self, event):
        region = self.tree.identify("region", event.x, event.y)
        if region != "cell":
            return
        col = self.tree.identify_column(event.x)
        row = self.tree.identify_row(event.y)
        if not row:
            return
        column_index = int(col.replace("#", ""))
        if column_index == 4:
            idx = int(row)
            self.delete_item_by_index(idx)

    def delete_item_by_index(self, idx):
        if not self.current_section:
            return
        total = len(self.sections[self.current_section])
        if total <= 1:
            self.status_var.set("Impossible de supprimer : une ligne obligatoire")
            return
        key = (self.current_section, idx)
        self.sections[self.current_section].pop(idx)
        if key in self.ratings:
            del self.ratings[key]
        new_ratings = {}
        for (sec, i), note in self.ratings.items():
            if sec != self.current_section:
                new_ratings[(sec, i)] = note
            else:
                if i > idx:
                    new_ratings[(sec, i - 1)] = note
                else:
                    new_ratings[(sec, i)] = note
        self.ratings = new_ratings
        self.refresh_items()
        self.status_var.set("Ligne supprimée")

    def search_item(self):
        query = self.search_var.get().strip()
        if not query.isdigit():
            messagebox.showwarning("Avertissement", "Entrez un ID numérique pour la recherche.")
            return
        target_id = int(query)
        results = []
        for sec, items in self.sections.items():
            for idx, (item_id, qty) in enumerate(items):
                if item_id == target_id:
                    results.append((sec, idx, qty))
        if not results:
            messagebox.showinfo("Résultat", f"Aucun item trouvé avec ID : {target_id}")
            return
        sec, idx, qty = results[0]
        self.current_section = sec
        for i, name in enumerate(self.section_listbox.get(0, tk.END)):
            if name == sec:
                self.section_listbox.selection_clear(0, tk.END)
                self.section_listbox.selection_set(i)
                self.section_listbox.see(i)
                break
        self.refresh_items()
        self.tree.selection_set(idx)
        self.tree.see(idx)
        self.status_var.set(f"Item trouvé dans [{sec}], index {idx}, quantité {qty}")

    # --- Lecture/Écriture fichier ---
    def parse_items_file(self, file_path):
        sections = {}
        ratings = {}
        current_section = None
        with open(file_path, "r", encoding="utf-8") as f:
            for line in f:
                line = line.rstrip()
                if not line:
                    continue
                header_match = re.match(r"^\[(.+?)\]$", line)
                if header_match:
                    current_section = header_match.group(1)
                    sections[current_section] = []
                elif current_section:
                    parts = re.split(r"\s*//\s*", line, maxsplit=1)
                    left = parts[0].strip()
                    note = parts[1].strip() if len(parts) > 1 else ""
                    vals = re.split(r"\s+", left)
                    if len(vals) >= 2:
                        try:
                            item_id = int(vals[0])
                            quantity = int(vals[1])
                            sections[current_section].append((item_id, quantity))
                            if note:
                                idx = len(sections[current_section]) - 1
                                ratings[(current_section, idx)] = note
                        except ValueError:
                            continue
        return sections, ratings

    def write_items_file(self, file_path, sections, ratings):
        with open(file_path, "w", encoding="utf-8") as f:
            for section, items in sections.items():
                f.write(f"[{section}]\n")
                for idx, (item_id, quantity) in enumerate(items):
                    note = ratings.get((section, idx), "")
                    if note:
                        f.write(f"\t{item_id}\t{quantity} // {note}\n")
                    else:
                        f.write(f"\t{item_id}\t{quantity}\n")
                f.write("\n")

# ──────────────────────────────────────────────────────────────────────────────
# MOB CREATE EDITOR (onglet "EditMobCreate")
# ──────────────────────────────────────────────────────────────────────────────

class InlineEditor:
    """
    Gestionnaire d'édition en ligne pour une cellule Treeview.
    """
    def __init__(self, tree, item, column, save_callback):
        self.tree = tree
        self.item = item
        self.column = column  # index integer de colonne (ex: 0 pour première colonne)
        self.save_callback = save_callback

        x, y, width, height = self.tree.bbox(item, f"#{column+1}")
        self.var = tk.StringVar()
        current = self.tree.set(item, self.tree["columns"][column])
        self.var.set(current)

        self.entry = tk.Entry(self.tree, textvariable=self.var, bg="white")
        # Placer l'Entry exactement sur la cellule
        self.entry.place(x=x, y=y, width=width, height=height)
        self.entry.focus()
        self.entry.select_range(0, tk.END)
        self.entry.bind("<Return>", self.on_save)
        self.entry.bind("<Escape>", self.on_cancel)
        self.entry.bind("<FocusOut>", self.on_cancel)

    def on_save(self, event=None):
        new_val = self.var.get().strip()
        self.entry.destroy()
        try:
            self.save_callback(self.item, self.column, new_val)
        except ValueError as e:
            messagebox.showerror("Erreur", str(e), parent=self.tree.master)
        finally:
            self.tree.focus_set()

    def on_cancel(self, event=None):
        self.entry.destroy()
        self.tree.focus_set()

class SetGroupEditor(tk.Frame):
    """
    Éditeur pour :
      - MobCreateSet_X → tableau (CountMobs, Percent) + DisableMap + suppr ligne
      - MobCreateGroup_Y → paramètres (SetMobMin, SetMobMax, SetMobRangeMin, SetMobRangeMax, SetMobFix)
        (SetMobFix : liste d’entiers, éditable en ligne)
    """
    def __init__(self, parent, app):
        super().__init__(parent, bg=DIR_BG)
        self.app = app
        self.pack(fill="both", expand=True, padx=8, pady=8)
        self.create_ui()

    def create_ui(self):
        # ===== MobCreateSet FRAME =====
        set_frame = tk.LabelFrame(
            self,
            text="MobCreateSet (CountMobs, Percent) + DisableMap",
            bg=DIR_BG,
            fg=LIGHT_FG
        )
        set_frame.pack(fill="both", padx=4, pady=4)

        cols_set = ["CountMobs", "Percent", "Suppr"]
        self.tree_set = ttk.Treeview(
            set_frame,
            columns=cols_set,
            show="headings",
            selectmode="browse",
            height=6
        )
        self.tree_set.heading("CountMobs", text="CountMobs")
        self.tree_set.heading("Percent", text="Percent")
        self.tree_set.heading("Suppr", text="")  # pas de texte dans le header
        self.tree_set.column("CountMobs", width=100, anchor="center")
        self.tree_set.column("Percent", width=100, anchor="center")
        self.tree_set.column("Suppr", width=50, anchor="center")
        self.tree_set.pack(fill="both", padx=4, pady=(4, 2))

        self.tree_set.bind("<Button-1>", self.on_tree_click)
        self.tree_set.bind("<Double-1>", self.on_tree_double_click)
        self.tree_set.bind("<<TreeviewSelect>>", lambda e: self.refresh_groups())

        disable_frame = tk.Frame(set_frame, bg=DIR_BG)
        disable_frame.pack(fill="x", padx=4, pady=(2, 4))
        tk.Label(disable_frame, text="DisableMap :", bg=DIR_BG, fg=LIGHT_FG).pack(side="left")
        self.disable_entry = tk.Entry(disable_frame, bg=DARK_FRAME, fg=LIGHT_FG, width=30)
        self.disable_entry.pack(side="left", padx=(4, 0))
        tk.Button(
            disable_frame,
            text="Modifier DisableMap",
            command=self.edit_disable_map,
            bg="#5e5e5e",
            fg=LIGHT_FG
        ).pack(side="left", padx=4)
        tk.Button(
            disable_frame,
            text="➕ Ajouter Ligne",
            command=self.add_column,
            bg="#5e5e5e",
            fg=LIGHT_FG
        ).pack(side="left", padx=16)

        # ===== MobCreateGroup FRAME =====
        grp_frame = tk.LabelFrame(
            self,
            text="MobCreateGroup (SetMobFix = liste d’entiers)",
            bg=DIR_BG,
            fg=LIGHT_FG
        )
        grp_frame.pack(fill="both", padx=4, pady=4)

        tk.Label(
            grp_frame,
            text="Double‐clic sur une cellule pour modifier directement (SetMobFix peut être '7,8,9')",
            bg=DIR_BG,
            fg=LIGHT_FG
        ).pack(anchor="w", padx=4, pady=(4, 0))

        cols_grp = ["GroupID", "SetMobMin", "SetMobMax", "SetMobRangeMin", "SetMobRangeMax", "SetMobFix"]
        self.tree_group = ttk.Treeview(
            grp_frame,
            columns=cols_grp,
            show="headings",
            selectmode="browse",
            height=6
        )
        for c in cols_grp:
            self.tree_group.heading(c, text=c)
            self.tree_group.column(c, width=100, anchor="center")

        self.tree_group.pack(fill="both", padx=4, pady=4)
        self.tree_group.bind("<Double-1>", self.on_group_double_click)

    def on_tree_click(self, event):
        """
        Si clic sur la colonne 'Suppr', supprimer la ligne correspondante,
        sauf s'il n'y a plus qu'une ligne restante.
        """
        region = self.tree_set.identify("region", event.x, event.y)
        if region != "cell":
            return
        col = self.tree_set.identify_column(event.x)
        if col != "#3":  # colonne 'Suppr' est #3
            return
        row_id = self.tree_set.identify_row(event.y)
        if not row_id:
            return
        sid = self.app.current_set
        if sid is None:
            return

        total_rows = len(self.tree_set.get_children())
        if total_rows <= 1:
            return

        idx = self.tree_set.index(row_id)
        count_mobs, _ = self.app.sets[sid]["column"][idx]

        # Supprimer la ligne du set
        del self.app.sets[sid]["column"][idx]
        self.tree_set.delete(row_id)
        self.update_suppr_column()

        # Si ce CountMobs n'apparaît plus dans aucun set, on peut retirer son bloc group
        still_used = any(
            (count_mobs == cm) for s in self.app.sets.values() for (cm, _) in s["column"]
        )
        if not still_used and count_mobs in self.app.groups:
            del self.app.groups[count_mobs]

        children = self.tree_set.get_children()
        if children:
            self.tree_set.selection_set(children[0])
        self.refresh_groups()

    def on_tree_double_click(self, event):
        """
        Double‐clic sur une cellule de MobCreateSet pour éditer en ligne CountMobs ou Percent.
        """
        row_id = self.tree_set.identify_row(event.y)
        col = self.tree_set.identify_column(event.x)
        if not row_id or col not in ("#1", "#2"):
            return
        col_index = int(col.replace("#", "")) - 1  # 0 pour CountMobs, 1 pour Percent
        InlineEditor(self.tree_set, row_id, col_index, self.save_set_cell)

    def save_set_cell(self, item, column, new_val):
        sid = self.app.current_set
        if sid is None:
            return

        idx = self.tree_set.index(item)
        old_count, old_pct = self.app.sets[sid]["column"][idx]

        if column == 0:
            # Édition de CountMobs
            try:
                cm = int(new_val)
            except ValueError:
                raise ValueError("CountMobs doit être un entier valide.")
            # Mettre à jour la ligne
            self.app.sets[sid]["column"][idx] = (cm, old_pct)

            # Supprimer l'ancien group s'il n'est plus utilisé
            still_old_used = any(
                (old_count == cm2) for s in self.app.sets.values() for (cm2, _) in s["column"]
            )
            if not still_old_used and old_count in self.app.groups:
                del self.app.groups[old_count]

            # Si nouveau CountMobs pas encore dans groups, l'ajouter
            if cm not in self.app.groups:
                self.app.groups[cm] = {"minmax": (0, 0), "range": (0, 0), "fix": []}

            new_count, new_pct = cm, old_pct

        else:
            # Édition de Percent
            try:
                pct = int(new_val)
            except ValueError:
                raise ValueError("Percent doit être un entier valide.")
            new_count, new_pct = old_count, pct
            self.app.sets[sid]["column"][idx] = (old_count, pct)

        sup = "❌" if len(self.tree_set.get_children()) > 1 else ""
        self.tree_set.item(item, values=(new_count, new_pct, sup))

    def update_suppr_column(self):
        """
        Si seule 1 ligne : masquer la croix. Sinon, afficher ❌.
        """
        children = self.tree_set.get_children()
        if len(children) <= 1:
            for row in children:
                cm, pct, _ = self.tree_set.item(row, "values")
                self.tree_set.item(row, values=(cm, pct, ""))
        else:
            for row in children:
                cm, pct, _ = self.tree_set.item(row, "values")
                self.tree_set.item(row, values=(cm, pct, "❌"))

    def refresh(self, set_id):
        """
        Recharge les données MobCreateSet_{set_id} dans tree_set + disable_map,
        puis invoque refresh_groups() pour afficher les MobCreateGroup.
        """
        self.tree_set.delete(*self.tree_set.get_children())
        if set_id is not None:
            rows = self.app.sets.get(set_id, {}).get("column", [])
            for (cm, pct) in rows:
                self.tree_set.insert("", tk.END, values=(cm, pct, "❌"))
            dm_list = self.app.sets[set_id].get("disable_map", [])
            self.disable_entry.delete(0, tk.END)
            self.disable_entry.insert(0, ",".join(str(x) for x in dm_list))
        else:
            self.disable_entry.delete(0, tk.END)

        self.update_suppr_column()
        children = self.tree_set.get_children()
        if children:
            self.tree_set.selection_set(children[0])
        self.refresh_groups()

    def refresh_groups(self):
        """
        Affiche tous les MobCreateGroup pour les CountMobs du set courant.
        """
        self.tree_group.delete(*self.tree_group.get_children())
        sid = self.app.current_set
        if sid is None:
            return

        # Récupérer la liste des CountMobs présents dans ce set
        cm_ids = {cm for (cm, _) in self.app.sets[sid].get("column", [])}
        for cm in sorted(cm_ids):
            if cm not in self.app.groups:
                # Initialiser si jamais absent
                self.app.groups[cm] = {"minmax": (0, 0), "range": (0, 0), "fix": []}
            gdata = self.app.groups[cm]
            mn, mx = gdata["minmax"]
            rmn, rmx = gdata["range"]
            fx_list = gdata["fix"]
            fx_str = ",".join(str(x) for x in fx_list) if isinstance(fx_list, list) else str(fx_list)
            self.tree_group.insert("", tk.END, values=(cm, mn, mx, rmn, rmx, fx_str))

    def add_column(self):
        """
        Ajoute une ligne automatiquement :
          - CountMobs = ID du set courant
          - Percent   = 100
        (sans fenêtre de saisie)
        """
        sid = self.app.current_set
        if sid is None:
            return

        lst = self.app.sets.setdefault(sid, {}).setdefault("column", [])
        if len(lst) >= 512:
            messagebox.showwarning("Limite atteinte", "Max 512 lignes atteint.", parent=self)
            return

        cm = sid
        pct = 100
        lst.append((cm, pct))
        self.tree_set.insert("", tk.END, values=(cm, pct, "❌"))
        self.update_suppr_column()
        self.tree_set.selection_set(self.tree_set.get_children()[-1])

        # Si ce CountMobs n'existe pas encore dans self.groups, on l'ajoute
        if cm not in self.app.groups:
            self.app.groups[cm] = {"minmax": (0, 0), "range": (0, 0), "fix": []}

        self.refresh_groups()

    def edit_disable_map(self):
        sid = self.app.current_set
        if sid is None:
            return
        dm_text = self.disable_entry.get().strip()
        if dm_text:
            try:
                dm_list = [int(x) for x in dm_text.split(",") if x.strip()]
            except ValueError:
                messagebox.showerror(
                    "Erreur",
                    "DisableMap doit être une liste d'entiers séparés par des virgules.",
                    parent=self
                )
                return
        else:
            dm_list = []
        self.app.sets[sid]["disable_map"] = dm_list
        messagebox.showinfo("Succès", f"DisableMap pour MobCreateSet_{sid} enregistré.", parent=self)

    def on_group_double_click(self, event):
        """
        Double‐clic sur une cellule de MobCreateGroup pour éditer en ligne,
        sauf sur la colonne GroupID.
        """
        row_id = self.tree_group.identify_row(event.y)
        col = self.tree_group.identify_column(event.x)
        if not row_id or col == "#1":
            return
        col_index = int(col.replace("#", "")) - 1  # 1 à 5 pour les champs modifiables
        InlineEditor(self.tree_group, row_id, col_index, self.save_group_cell)

    def save_group_cell(self, item, column, new_val):
        vals = self.tree_group.item(item, "values")
        gid = int(vals[0])

        # Si on édite SetMobFix (colonne index = 5), on attend "7,8,9" ou "5"
        if column == 5:
            parts = [x.strip() for x in new_val.split(",") if x.strip()]
            try:
                fx_list = [int(x) for x in parts]
            except ValueError:
                raise ValueError("SetMobFix doit être une liste d'entiers séparés par des virgules.")
            self.app.groups[gid]["fix"] = fx_list
        else:
            # pour les autres colonnes (1 à 4), ce sont toujours des entiers simples
            try:
                iv = int(new_val)
            except ValueError:
                raise ValueError("La valeur doit être un entier valide.")

            mn, mx = self.app.groups[gid]["minmax"]
            rmn, rmx = self.app.groups[gid]["range"]
            fx_list = self.app.groups[gid]["fix"]

            if column == 1:
                mn = iv
            elif column == 2:
                mx = iv
            elif column == 3:
                rmn = iv
            elif column == 4:
                rmx = iv

            self.app.groups[gid]["minmax"] = (mn, mx)
            self.app.groups[gid]["range"] = (rmn, rmx)

        # Mise à jour de l’affichage
        mn, mx = self.app.groups[gid]["minmax"]
        rmn, rmx = self.app.groups[gid]["range"]
        fx_list = self.app.groups[gid]["fix"]
        fx_str = ",".join(str(x) for x in fx_list) if isinstance(fx_list, list) else str(fx_list)
        self.tree_group.item(item, values=(gid, mn, mx, rmn, rmx, fx_str))

class MobCreateFrame(tk.Frame):
    """
    Frame englobant l’éditeur MobCreateSet / MobCreateGroup,
    au lieu de créer une fenêtre Tk distincte.
    """
    def __init__(self, parent):
        super().__init__(parent, bg=DIR_BG)
        self.sets = {}         # self.sets[set_id] = {"column": [(CountMobs,Percent), ...], "disable_map": [...]}
        self.groups = {}       # self.groups[group_id] = {"minmax": (mn,mx), "range": (rmn,rmx), "fix": [...]}
        self.current_set = None
        self.filename = None

        self._setup_style()
        self._build_ui()

    def _setup_style(self):
        style = ttk.Style(self)
        style.theme_use("clam")
        style.configure(
            "Treeview",
            background=DARK_FRAME,
            fieldbackground=DARK_FRAME,
            foreground=LIGHT_FG,
            bordercolor=DARK_FRAME,
            borderwidth=0,
        )
        style.map(
            "Treeview",
            background=[("selected", "#5e5e5e")],
            foreground=[("selected", LIGHT_FG)],
        )

    def _build_ui(self):
        main_pane = ttk.Panedwindow(self, orient=tk.HORIZONTAL)
        main_pane.pack(fill=tk.BOTH, expand=True, padx=8, pady=8)

        # ===== Liste des Sets à gauche =====
        left_frame = tk.Frame(main_pane, bg=DIR_BG, width=250)
        main_pane.add(left_frame, weight=1)

        tk.Label(
            left_frame,
            text="MobCreateSet IDs",
            bg=DIR_BG,
            fg=LIGHT_FG,
            font=("Arial", 12, "bold"),
        ).pack(anchor="w", pady=(4, 8), padx=4)

        self.list_sets = tk.Listbox(
            left_frame,
            bg=DARK_FRAME,
            fg=LIGHT_FG,
            selectbackground="#5e5e5e",
            highlightthickness=0,
            exportselection=False,
        )
        self.list_sets.pack(fill=tk.BOTH, expand=True, padx=4, pady=(0, 4))
        self.list_sets.bind("<<ListboxSelect>>", self.on_select_set)

        btn_frame = tk.Frame(left_frame, bg=DIR_BG)
        btn_frame.pack(fill="x", pady=4)
        tk.Button(
            btn_frame, text="➕ Ajouter Set", command=self.add_set, bg="#5e5e5e", fg=LIGHT_FG
        ).pack(side="left", fill="x", expand=True, padx=2)
        tk.Button(
            btn_frame, text="❌ Supprimer Set", command=self.delete_set, bg="#5e5e5e", fg=LIGHT_FG
        ).pack(side="left", fill="x", expand=True, padx=2)

        # ===== Boutons Charger / Sauvegarder =====
        tk.Button(
            left_frame,
            text="ðﾟﾓﾂ Charger fichier",
            command=self.load_from_file,
            bg="#5e5e5e",
            fg=LIGHT_FG
        ).pack(fill="x", padx=4, pady=(4, 0))
        tk.Button(
            left_frame,
            text="ðﾟﾒﾾ Sauvegarder Fichier",
            command=self.save_to_file,
            bg="#5e5e5e",
            fg=LIGHT_FG
        ).pack(fill="x", padx=4, pady=(4, 4))

        # ===== Éditeur à droite =====
        right_frame = tk.Frame(main_pane, bg=DIR_BG)
        main_pane.add(right_frame, weight=3)
        self.editor = SetGroupEditor(right_frame, self)

    def populate_list(self):
        self.list_sets.delete(0, tk.END)
        for sid in sorted(self.sets.keys()):
            self.list_sets.insert(tk.END, f"MobCreateSet_{sid}")

    def on_select_set(self, event):
        sel = self.list_sets.curselection()
        if not sel:
            self.current_set = None
            self.editor.refresh(None)
            return

        text = self.list_sets.get(sel[0])
        sid = int(text.split("_")[1])
        self.current_set = sid

        if sid not in self.sets:
            self.sets[sid] = {"column": [], "disable_map": []}

        self.editor.refresh(sid)

    def add_set(self):
        if self.sets:
            new_id = max(self.sets.keys()) + 1
        else:
            new_id = 1

        # Préremplir MobCreateSet : une seule ligne (CountMobs=new_id, Percent=100)
        self.sets[new_id] = {
            "column": [(new_id, 100)],
            "disable_map": []
        }
        # Créer un bloc MobCreateGroup par défaut pour ce même ID
        self.groups[new_id] = {"minmax": (0, 0), "range": (0, 0), "fix": []}

        self.populate_list()
        idx = list(sorted(self.sets.keys())).index(new_id)
        self.list_sets.selection_clear(0, tk.END)
        self.list_sets.selection_set(idx)
        self.on_select_set(None)

    def delete_set(self):
        sid = self.current_set
        if sid is None:
            return
        if messagebox.askyesno("Confirmer", f"Supprimer MobCreateSet_{sid} ?", parent=self):
            # Supprimer tous les CountMobs de ce set
            for (cm, _) in self.sets[sid]["column"]:
                # Si ce cm n'est plus utilisé ailleurs, supprimer son bloc group
                still_used = any(
                    (cm == cm2) for s in self.sets.values() for (cm2, _) in s["column"]
                )
                if not still_used and cm in self.groups:
                    del self.groups[cm]
            del self.sets[sid]

            self.current_set = None
            self.populate_list()
            self.editor.refresh(None)

    def load_from_file(self):
        path = filedialog.askopenfilename(
            defaultextension=".ini",
            filetypes=[("Fichiers INI", "*.ini"), ("Tous fichiers", "*.*")],
            title="Ouvrir MobCreate.ini",
        )
        if not path:
            return
        try:
            with open(path, "r", encoding="utf-8") as f:
                lines = f.readlines()
        except Exception as e:
            messagebox.showerror("Erreur", f"Impossible de charger : {e}", parent=self)
            return

        self.sets.clear()
        self.groups.clear()

        set_header = re.compile(r"^\[MobCreateSet_(\d+)\]")
        group_header = re.compile(r"^\[MobCreateGroup_(\d+)\]")
        col_inline_pattern = re.compile(r"\(\s*(\d+)\s*,\s*(\d+)\s*\)")
        disable_pattern = re.compile(r"\(\s*([\d,]+)\s*\)")
        minmax_pattern = re.compile(r"\(\s*(\d+)\s*~\s*(\d+)\s*\)")
        fix_pattern = re.compile(r"\(\s*([\d,]+)\s*\)")

        i = 0
        while i < len(lines):
            line = lines[i].strip()
            if not line or line.startswith(";"):
                i += 1
                continue

            # --- Section MobCreateSet_X ---
            m_set = set_header.match(line)
            if m_set:
                sid = int(m_set.group(1))
                self.sets[sid] = {"column": [], "disable_map": []}
                i += 1
                while i < len(lines):
                    ln = lines[i].strip()
                    if ln.startswith("[MobCreateSet_") or ln.startswith("[MobCreateGroup_"):
                        break
                    if ln.startswith("[MobCreateColumn]"):
                        parts = ln.split("=", 1)
                        if len(parts) == 2:
                            inline = parts[1]
                            for cm_str, p_str in col_inline_pattern.findall(inline):
                                cm, p = int(cm_str), int(p_str)
                                self.sets[sid]["column"].append((cm, p))
                        j = i + 1
                        while j < len(lines) and lines[j].strip().startswith("(") and ',' in lines[j]:
                            ln2 = lines[j].strip()
                            for cm_str, p_str in col_inline_pattern.findall(ln2):
                                cm, p = int(cm_str), int(p_str)
                                self.sets[sid]["column"].append((cm, p))
                            j += 1
                        i = j
                        continue
                    if ln.startswith("[MobCreateDiableMap]"):
                        parts = ln.split("=", 1)
                        if len(parts) == 2:
                            val = parts[1].strip()
                            dm_match = disable_pattern.search(val)
                            if dm_match:
                                dm_list = [int(x) for x in dm_match.group(1).split(",")]
                                self.sets[sid]["disable_map"] = dm_list
                    i += 1
                continue

            # --- Section MobCreateGroup_Y ---
            m_group = group_header.match(line)
            if m_group:
                gid = int(m_group.group(1))
                # Initialiser fix comme liste vide
                self.groups[gid] = {"minmax": (0, 0), "range": (0, 0), "fix": []}
                i += 1
                while i < len(lines):
                    ln = lines[i].strip()
                    if ln.startswith("[MobCreateSet_") or ln.startswith("[MobCreateGroup_"):
                        break
                    if ln.startswith("[SetMobMinMax]"):
                        parts = ln.split("=", 1)
                        if len(parts) == 2:
                            mm = minmax_pattern.search(parts[1].strip())
                            if mm:
                                self.groups[gid]["minmax"] = (int(mm.group(1)), int(mm.group(2)))
                    elif ln.startswith("[SetMobRange]"):
                        parts = ln.split("=", 1)
                        if len(parts) == 2:
                            rg = minmax_pattern.search(parts[1].strip())
                            if rg:
                                self.groups[gid]["range"] = (int(rg.group(1)), int(rg.group(2)))
                    elif ln.startswith("[SetMobFix]"):
                        parts = ln.split("=", 1)
                        if len(parts) == 2:
                            fx_match = fix_pattern.search(parts[1].strip())
                            if fx_match:
                                lst = [int(x) for x in fx_match.group(1).split(",")]
                                self.groups[gid]["fix"] = lst
                    i += 1
                continue

            i += 1

        self.populate_list()
        if self.list_sets.size() > 0:
            self.list_sets.selection_set(0)
            self.on_select_set(None)
        self.filename = path
        messagebox.showinfo("Importé", f"INI chargé depuis : {path}", parent=self)

    def save_to_file(self):
        # Si aucun chemin de fichier n'a encore été défini, on demande « Enregistrer sous... »
        if not self.filename:
            path = filedialog.asksaveasfilename(
                defaultextension=".ini",
                filetypes=[("Fichiers INI", "*.ini"), ("Tous fichiers", "*.*")],
                title="Enregistrer MobCreate.ini",
            )
            if not path:
                return
            self.filename = path

        try:
            with open(self.filename, "w", encoding="utf-8") as f:
                # 1) Écrire d'abord toutes les sections [MobCreateSet_<sid>]
                for sid in sorted(self.sets.keys()):
                    sdata = self.sets[sid]

                    # En-tête de section
                    f.write(f"[MobCreateSet_{sid}]\n")

                    # Bloc MobCreateColumn
                    if sdata["column"]:
                        f.write("[MobCreateColumn] =\n")
                        for (cm, pct) in sdata["column"]:
                            f.write(f"({cm},{pct})\n")
                    else:
                        f.write("[MobCreateColumn] = ()\n")

                    # Bloc MobCreateDiableMap
                    dm_list = sdata.get("disable_map", [])
                    if dm_list:
                        f.write(f"[MobCreateDiableMap] = ({','.join(str(x) for x in dm_list)})\n\n")
                    else:
                        f.write("[MobCreateDiableMap] = ()\n\n")

                # 2) Après avoir listé toutes les MobCreateSet, on met une ligne vide
                f.write("\n")

                # 3) Maintenant, écrire tous les blocs [MobCreateGroup_<gid>] triés
                used_group_ids = set()
                for sdata in self.sets.values():
                    for (cm, _) in sdata["column"]:
                        used_group_ids.add(cm)

                for gid in sorted(used_group_ids):
                    gdata = self.groups.get(gid, {"minmax": (0, 0), "range": (0, 0), "fix": []})
                    mn, mx = gdata["minmax"]
                    rmn, rmx = gdata["range"]
                    fx_list = gdata["fix"]
                    fx_str = ",".join(str(x) for x in fx_list) if isinstance(fx_list, list) else str(fx_list)

                    f.write(f"[MobCreateGroup_{gid}]\n")
                    f.write(f"[SetMobMinMax] = ({mn} ~ {mx})\n")
                    f.write(f"[SetMobRange] = ({rmn} ~ {rmx})\n")
                    f.write(f"[SetMobFix] = ({fx_str})\n\n")

            messagebox.showinfo("Enregistré", f"INI enregistré sous : {self.filename}", parent=self)

        except Exception as e:
            messagebox.showerror("Erreur", f"Impossible d’enregistrer : {e}", parent=self)
# ──────────────────────────────────────────────────────────────────────────────
# (1) Les patterns pour coloration syntaxique Lua + Shaiya
# ──────────────────────────────────────────────────────────────────────────────
class SyntaxPatterns:
    LUA_KEYWORDS = [
        'and', 'break', 'do', 'else', 'elseif', 'end', 'false', 'for', 'function',
        'if', 'in', 'local', 'nil', 'not', 'or', 'repeat', 'return', 'then',
        'true', 'until', 'while'
    ]
    SHAIYA_KEYWORDS = [
        'on_login', 'on_enter_map', 'on_item_effect', 'on_can_recreate',
        'on_can_equip', 'on_attackable', 'on_attacked', 'on_return_home',
        'on_normal_reset', 'on_death', 'on_move_end', 'while_combat',
        'LuaUpdateInsZonePortal', 'send_notice', 'add_gold', 'get_map_id',
        'spawn_mob', 'remove_mob', 'give_item', 'take_item', 'set_hp',
        'get_hp', 'set_position', 'get_position', 'set_level', 'get_level'
    ]
    PATTERN_COMMENT = re.compile(r'--\[\[.*?\]\]|--.*?$', re.MULTILINE | re.DOTALL)
    PATTERN_STRING  = re.compile(r"(\".*?\"|\'.*?\')", re.DOTALL)
    PATTERN_NUMBER  = re.compile(r"\b\d+(\.\d+)?\b")
    PATTERN_KEYWORD = re.compile(r"\b(" + r"|".join(LUA_KEYWORDS) + r")\b")
    PATTERN_SHAIYA  = re.compile(r"\b(" + r"|".join(SHAIYA_KEYWORDS) + r")\b")


# ──────────────────────────────────────────────────────────────────────────────
# (2) La classe LuaExplorerFrame (Frame à ajouter dans le Notebook)
# ──────────────────────────────────────────────────────────────────────────────
class LuaExplorerFrame(tk.Frame):
    """
    Frame qui contient l’« Exploreur Lua » complet, prêt à être ajouté dans un Notebook.
    """
    def __init__(self, parent):
        super().__init__(parent, bg="#1e1e1e")
        self.folder_path = None
        self.current_file = None
        self.file_lines = []
        self.functions = {}  # { "sig_start_end": [start, end, bool_folded], ... }

        # Dictionnaire des fonctions Shaiya (exemple statique)
        self.shaiya_functions = {
            "Épisode 1": [
                "on_login(dwTime, dwCharID)",
                "on_enter_map(dwTime, dwMapID)",
                "on_item_effect(dwTime, dwCharID, dwItemID)"
            ],
            "Épisode 2": [
                "on_can_recreate(dwTime, dwCharID)",
                "on_can_equip(dwTime, dwCharID, dwItemID)",
                "on_attackable(dwTime, dwCharID)"
            ],
            "Épisode 3": [
                "on_attacked(dwTime, dwCharID)",
                "on_return_home(dwTime, dwAttackedCount)",
                "on_normal_reset(dwTime)"
            ],
            "Épisode 4": [
                "on_death(dwTime, dwAttackedCount)",
                "on_move_end(dwTime)",
                "while_combat(dwTime, dwHPPercent, dwAttackedCount)"
            ]
        }
        self.flat_shaiya_funcs = [
            sig for funcs in self.shaiya_functions.values() for sig in funcs
        ]

        self._init_theme()
        self._create_widgets()
        self._configure_tags()
        self._bind_events()

    def _init_theme(self):
        # palette sombre cohérente avec le reste de l'application
        self.theme = {
            'bg': "#1e1e1e", 'fg': "#d4d4d4",
            'gutter_bg': "#2d2d2d", 'gutter_fg': "#858585",
            'keyword': "#569CD6", 'comment': "#6A9955",
            'string': "#CE9178", 'number': "#B5CEA8", 'shaiya': "#D16D9E",
            'select_bg': "#264F78", 'select_fg': "#FFFFFF",
            'button_bg': "#3c3c3c", 'button_fg': "#d4d4d4",
            'status_bg': "#007ACC", 'status_fg': "#FFFFFF"
        }

    def _create_widgets(self):
        # ─── PanedWindow principal (Explorateur | Centre | Description) ───
        main_paned = ttk.Panedwindow(self, orient=tk.HORIZONTAL)
        main_paned.pack(fill=tk.BOTH, expand=True)

        # ---- (A) Explorateur de fichiers à gauche ----
        explorer_frame = tk.Frame(main_paned, bg=self.theme['bg'])
        main_paned.add(explorer_frame, weight=1)

        self.file_listbox = tk.Listbox(
            explorer_frame,
            bg=self.theme['gutter_bg'], fg=self.theme['gutter_fg'],
            selectbackground=self.theme['select_bg'], selectforeground=self.theme['select_fg'],
            bd=0, highlightthickness=0
        )
        self.file_listbox.pack(fill=tk.BOTH, expand=True, padx=(5,0), pady=(5,5))
        self.file_listbox.bind('<Double-Button-1>', self._on_file_double_click)

        btn_frame = tk.Frame(explorer_frame, bg=self.theme['bg'])
        btn_frame.pack(side=tk.BOTTOM, fill=tk.X, padx=5, pady=5)

        tk.Button(
            btn_frame, text="Charger Dossier", command=self.open_folder,
            bg=self.theme['button_bg'], fg=self.theme['button_fg'],
            activebackground=self.theme['select_bg'], activeforeground=self.theme['fg'],
            bd=0, padx=5, pady=4
        ).pack(fill=tk.X, pady=(0,5))

        tk.Button(
            btn_frame, text="Enregistrer", command=self.save_current_file,
            bg=self.theme['button_bg'], fg=self.theme['button_fg'],
            activebackground=self.theme['select_bg'], activeforeground=self.theme['fg'],
            bd=0, padx=5, pady=4
        ).pack(fill=tk.X, pady=(0,5))

        tk.Button(
            btn_frame, text="Nouveau Lua", command=self.create_new_lua,
            bg=self.theme['button_bg'], fg=self.theme['button_fg'],
            activebackground=self.theme['select_bg'], activeforeground=self.theme['fg'],
            bd=0, padx=5, pady=4
        ).pack(fill=tk.X, pady=(0,5))

        tk.Button(
            btn_frame, text="Supprimer Lua", command=self.delete_selected_lua,
            bg=self.theme['button_bg'], fg=self.theme['button_fg'],
            activebackground=self.theme['select_bg'], activeforeground=self.theme['fg'],
            bd=0, padx=5, pady=4
        ).pack(fill=tk.X)

        # ---- (B) Panneau central (Vertical) : Tableau fonctions + Éditeur ----
        center_paned = ttk.Panedwindow(main_paned, orient=tk.VERTICAL)
        main_paned.add(center_paned, weight=4)

        # (B1) Tableau des fonctions détectées (en haut)
        func_container = tk.Frame(center_paned, bg=self.theme['bg'], bd=1, relief=tk.SUNKEN)
        center_paned.add(func_container, weight=1)

        tk.Label(
            func_container, text="Fonctions détectées",
            font=("Consolas", 10, "bold"), bg=self.theme['bg'], fg=self.theme['fg']
        ).pack(pady=(5,5))

        cols = ("Signature", "Début", "Fin", "Suppr")
        style = ttk.Style(func_container)
        style.theme_use("clam")
        style.configure("Func.Treeview",
                        background=self.theme['gutter_bg'],
                        foreground=self.theme['gutter_fg'],
                        fieldbackground=self.theme['gutter_bg'])
        style.map("Func.Treeview",
                  background=[("selected", self.theme['select_bg'])],
                  foreground=[("selected", self.theme['select_fg'])])

        self.func_table = ttk.Treeview(
            func_container,
            columns=cols,
            show="headings",
            height=8,
            style="Func.Treeview"
        )
        self.func_table.heading("Signature", text="Signature")
        self.func_table.heading("Début",     text="Début")
        self.func_table.heading("Fin",       text="Fin")
        self.func_table.heading("Suppr",     text="ðﾟﾗﾙ")
        self.func_table.column("Signature",  width=200, anchor="w")
        self.func_table.column("Début",      width=60,  anchor="center", stretch=False)
        self.func_table.column("Fin",        width=60,  anchor="center", stretch=False)
        self.func_table.column("Suppr",      width=40,  anchor="center", stretch=False)
        self.func_table.pack(fill=tk.BOTH, expand=True, padx=5, pady=(0,5))
        self.func_table.bind("<ButtonRelease-1>", self._on_func_table_click)

        # (B2) Éditeur de code + gutter + panneau "Ajouter Fonction" (en bas)
        editor_container = tk.Frame(center_paned, bg=self.theme['bg'], bd=1, relief=tk.SUNKEN)
        center_paned.add(editor_container, weight=3)

        # ---- Éditeur de texte principal (largeur augmentée de 20 colonnes) ----
        self.editor_text = tk.Text(
            editor_container,
            wrap='none',
            undo=True,
            width=0,  # ← augmenter la largeur
            bg=self.theme['bg'], fg=self.theme['fg'],
            insertbackground=self.theme['fg'],
            selectbackground=self.theme['select_bg'], selectforeground=self.theme['select_fg'],
            bd=0, highlightthickness=0
        )
        self.editor_text.pack(fill=tk.BOTH, expand=True, side=tk.LEFT, pady=5)
        self.editor_text.bind("<ButtonRelease-1>", self._on_editor_click)

        # ---- Gutter (numéros de ligne) à droite de l’éditeur ----
        self.editor_gutter = tk.Text(
            editor_container,
            width=4, padx=4, takefocus=0, border=0,
            bg=self.theme['gutter_bg'], fg=self.theme['gutter_fg'],
            state='disabled', wrap='none'
        )
        self.editor_gutter.pack(side=tk.LEFT, fill=tk.Y, pady=5)

        # ---- Panneau “Ajouter Fonction” (Shaiya) à droite du gutter ----
        add_func_frame = tk.Frame(editor_container, bg=self.theme['bg'])
        add_func_frame.pack(side=tk.LEFT, fill=tk.Y, padx=(2,2), pady=5)

        tk.Label(
            add_func_frame, text="Ajouter Fonction", font=("Consolas", 9, "bold"),
            bg=self.theme['bg'], fg=self.theme['fg']
        ).pack(pady=(2,2))

        self.func_listbox = tk.Listbox(
            add_func_frame,
            bg=self.theme['gutter_bg'], fg=self.theme['gutter_fg'],
            selectbackground=self.theme['select_bg'], selectforeground=self.theme['select_fg'],
            exportselection=False,
            width=30,
            height=15
        )
        self.func_listbox.pack(fill=tk.BOTH, expand=True, pady=(0,5))
        self.func_listbox.bind("<<ListboxSelect>>", self._on_shaiya_func_click)

        for sig in self.flat_shaiya_funcs:
            self.func_listbox.insert(tk.END, sig)

        # ---- Scrollbar verticale pour l’éditeur (seule scrollbar restante) ----
        editor_scroll = tk.Scrollbar(editor_container, command=self._editor_on_scroll, bg=self.theme['bg'])
        editor_scroll.pack(fill=tk.Y, side=tk.LEFT, pady=5)
        self.editor_text.config(yscrollcommand=self._editor_on_yscroll)

        # Tag "folded" pour pliage de fonctions (elide=True masque le texte)
        self.editor_text.tag_configure("folded", elide=True)

        # Barre d'état en bas du Frame LuaExplorerFrame
        self.status_bar = tk.Label(
            self, text="Aucun dossier chargé", bd=1, relief=tk.SUNKEN,
            anchor='w', bg=self.theme['status_bg'], fg=self.theme['status_fg']
        )
        self.status_bar.pack(side=tk.BOTTOM, fill=tk.X)

        # ---- (C) Panneau Description à l’extrême droite ----
        desc_frame = tk.Frame(main_paned, bg=self.theme['bg'], bd=1, relief=tk.SUNKEN)
        main_paned.add(desc_frame, weight=2)

        tk.Label(
            desc_frame, text="Description des fonctionnalités",
            font=("Consolas", 10, "bold"),
            bg=self.theme['bg'], fg=self.theme['fg']
        ).pack(pady=(5,5), padx=5)

        self.desc_text = tk.Text(
            desc_frame,
            wrap='word',
            bg=self.theme['gutter_bg'], fg=self.theme['gutter_fg'],
            insertbackground=self.theme['gutter_fg'],
            bd=0, highlightthickness=0
        )
        self.desc_text.pack(fill=tk.BOTH, expand=True, padx=5, pady=5)
        self.desc_text.insert(tk.END, self._get_description_content())
        self.desc_text.config(state='disabled')

    # ──────────────────────────────────────────────────────────────────────────
    # Méthodes de gestion de l’explorateur et des fichiers .lua
    # ──────────────────────────────────────────────────────────────────────────
    def open_folder(self):
        """
        Ouvre une boîte de dialogue pour sélectionner un dossier,
        puis liste tous les fichiers .lua dans le Listbox.
        """
        folder = filedialog.askdirectory()
        if not folder:
            return
        self.folder_path = folder
        self.file_listbox.delete(0, tk.END)
        try:
            files = sorted(os.listdir(folder), key=str.lower)
            lua_files = [f for f in files if f.lower().endswith('.lua')]
            for fname in lua_files:
                self.file_listbox.insert(tk.END, fname)
            self.status_bar.config(text=f"Dossier : {os.path.basename(folder)} – {len(lua_files)} fichier(s) .lua")
        except Exception as e:
            messagebox.showerror("Erreur", f"Impossible de lister le dossier :\n{e}")
            return

        # Réinitialiser l’éditeur
        self.current_file = None
        self.editor_text.delete("1.0", tk.END)
        self.editor_gutter.config(state='normal')
        self.editor_gutter.delete("1.0", tk.END)
        self.editor_gutter.config(state='disabled')
        self.func_table.delete(*self.func_table.get_children())
        self.functions.clear()

    def _on_file_double_click(self, event):
        """
        Ouvre le fichier .lua double-cliqué dans le Listbox.
        """
        sel = self.file_listbox.curselection()
        if not sel:
            return
        idx = sel[0]
        name = self.file_listbox.get(idx)
        fullpath = os.path.join(self.folder_path, name)
        self._load_file(fullpath, name)

    def _load_file(self, path: str, display_name: str):
        """
        Charge le contenu du fichier (UTF-8 puis Latin-1 si échec),
        affiche-le dans l’éditeur, reconstruit la table des fonctions,
        et plie par défaut tous les blocs.
        """
        try:
            with open(path, 'r', encoding='utf-8') as f:
                content = f.read()
        except UnicodeDecodeError:
            try:
                with open(path, 'r', encoding='latin-1') as f:
                    content = f.read()
            except Exception as e:
                messagebox.showerror("Erreur", f"Impossible de lire :\n{e}")
                return
        except Exception as e:
            messagebox.showerror("Erreur", f"Impossible d’ouvrir :\n{e}")
            return

        self.current_file = display_name
        self.file_lines = content.splitlines()

        # Afficher dans l’éditeur
        self.editor_text.delete("1.0", tk.END)
        self.editor_text.insert("1.0", content)
        self.editor_text.edit_modified(False)
        self._update_editor_gutter()
        self._highlight_editor_syntax()

        # Reconstruire la table des fonctions et plier
        self._populate_function_table()
        self._fold_all_functions()

        self.status_bar.config(text=f"{display_name} – {len(self.functions)} fonction(s) détectée(s)")

    def save_current_file(self):
        """
        Sauvegarde le fichier actuellement ouvert (current_file) si défini.
        """
        if not self.current_file or not self.folder_path:
            messagebox.showwarning("Sauvegarde", "Aucun fichier ouvert à enregistrer.")
            return
        path = os.path.join(self.folder_path, self.current_file)
        try:
            content = self.editor_text.get("1.0", tk.END)
            with open(path, 'w', encoding='utf-8') as f:
                f.write(content)
            messagebox.showinfo("Sauvegarde", f"Fichier enregistré :\n{self.current_file}")
        except Exception as e:
            messagebox.showerror("Erreur d’enregistrement", f"Impossible d’enregistrer :\n{e}")

    def create_new_lua(self):
        """
        Crée automatiquement un nouveau fichier .lua vide dans le dossier courant
        (nommé untitled1.lua, untitled2.lua, … sans fenêtre supplémentaire).
        """
        if not self.folder_path:
            messagebox.showwarning("Nouveau Lua", "Chargez d’abord un dossier pour créer un fichier.")
            return
        base = "untitled"
        idx = 1
        while True:
            name = f"{base}{idx}.lua"
            full = os.path.join(self.folder_path, name)
            if not os.path.exists(full):
                try:
                    with open(full, 'w', encoding='utf-8') as f:
                        f.write("")  # vide
                    self.file_listbox.insert(tk.END, name)
                    self.file_listbox.selection_clear(0, tk.END)
                    self.file_listbox.selection_set(tk.END)
                    self.file_listbox.see(tk.END)
                    self._load_file(full, name)
                    self.status_bar.config(text=f"Nouveau fichier créé : {name}")
                except Exception as e:
                    messagebox.showerror("Erreur", f"Impossible de créer le fichier :\n{e}")
                break
            idx += 1

    def delete_selected_lua(self):
        """
        Supprime le fichier .lua sélectionné dans l’explorateur (avec confirmation).
        Retire le fichier du disque ET de la liste. Si c’est le fichier ouvert,
        vide l’éditeur.
        """
        sel = self.file_listbox.curselection()
        if not sel:
            messagebox.showwarning("Supprimer Lua", "Aucun fichier sélectionné à supprimer.")
            return
        idx = sel[0]
        name = self.file_listbox.get(idx)
        confirm = messagebox.askyesno(
            "Supprimer Lua",
            f"Voulez-vous vraiment supprimer le fichier :\n{name} ?"
        )
        if not confirm:
            return
        full = os.path.join(self.folder_path, name)
        try:
            os.remove(full)
            self.file_listbox.delete(idx)
            if self.current_file == name:
                # Vider l’éditeur
                self.current_file = None
                self.editor_text.delete("1.0", tk.END)
                self.editor_gutter.config(state='normal')
                self.editor_gutter.delete("1.0", tk.END)
                self.editor_gutter.config(state='disabled')
                self.func_table.delete(*self.func_table.get_children())
                self.functions.clear()
                self.status_bar.config(text="Aucun fichier ouvert")
        except Exception as e:
            messagebox.showerror("Erreur", f"Impossible de supprimer :\n{e}")

    # ──────────────────────────────────────────────────────────────────────────
    # Méthodes de gestion du tableau de fonctions (détection, pliage, suppression)
    # ──────────────────────────────────────────────────────────────────────────
    def _populate_function_table(self):
        """
        Analyse `self.file_lines` pour détecter toutes les fonctions racine (blocs imbriqués),
        puis remplit `self.func_table`. Colonne “Suppr” reçoit “ðﾟﾗﾙ”.
        """
        self.func_table.delete(*self.func_table.get_children())
        self.functions.clear()

        lines = self.file_lines
        total = len(lines)
        idx = 0
        # On considère : début de bloc = function … / if … then / while … do / for … do / repeat
        re_start = re.compile(r'^\s*(function\b|if\b.*\bthen\b|while\b.*\bdo\b|for\b.*\bdo\b|repeat\b)')
        re_end   = re.compile(r'^\s*(end\b|until\b)')
        func_pat = re.compile(r'^\s*function\s+([A-Za-z0-9_\.]+)\s*\((.*?)\)')

        while idx < total:
            line = lines[idx]
            m = func_pat.match(line)
            if m:
                start = idx
                depth = 0
                # Remonter le bloc imbriqué jusqu’à depth=0
                for j in range(idx, total):
                    l = lines[j]
                    stripped = l.strip()
                    if stripped.startswith('--'):
                        continue
                    if re_start.match(l):
                        depth += 1
                    elif re_end.match(l):
                        depth -= 1
                        if depth == 0:
                            end = j
                            break
                else:
                    end = start

                sig = f"{m.group(1)}({m.group(2).strip()})"
                key = f"{sig}_{start+1}_{end+1}"
                self.functions[key] = [start+1, end+1, False]
                self.func_table.insert(
                    "", tk.END,
                    values=(sig, start+1, end+1, "ðﾟﾗﾙ")
                )
                idx = end + 1
            else:
                idx += 1

    def _fold_all_functions(self):
        """
        Plie (cache) toutes les fonctions détectées en appliquant le tag "folded".
        """
        for key, (start, end, _) in self.functions.items():
            start_idx = f"{start}.0"
            end_idx   = f"{end+1}.0"
            self.editor_text.tag_add("folded", start_idx, end_idx)
            self.functions[key][2] = True

    def _toggle_fold(self, sig: str, start_line: int, end_line: int):
        """
        Plie/déplie le bloc de fonction spécifié par sig et ses bornes.
        """
        key = f"{sig}_{start_line}_{end_line}"
        if key not in self.functions:
            return
        start, end, folded = self.functions[key]
        start_idx = f"{start}.0"
        end_idx   = f"{end+1}.0"
        if folded:
            # Déplier
            self.editor_text.tag_remove("folded", start_idx, end_idx)
            self.functions[key][2] = False
            # Sélectionner la ligne de début
            self.editor_text.tag_remove("sel", "1.0", tk.END)
            self.editor_text.tag_add("sel", start_idx, f"{start}.end")
            self.editor_text.mark_set("insert", start_idx)
            self.editor_text.see(start_idx)
        else:
            # Replier
            self.editor_text.tag_add("folded", start_idx, end_idx)
            self.functions[key][2] = True

    def _on_func_table_click(self, event):
        """
        Si clic dans la colonne “Suppr” (col #4), supprime la fonction correspondante.
        Sinon, toggler le pliage/dépliage, et garder la sélection active.
        """
        region = self.func_table.identify("region", event.x, event.y)
        if region != "cell":
            return
        column = self.func_table.identify_column(event.x)
        row    = self.func_table.identify_row(event.y)
        if not row:
            return
        item = row
        values = self.func_table.item(item, "values")
        sig, start_str, end_str, _ = values
        start_line = int(start_str)
        end_line   = int(end_str)
        if column == "#4":  # Suppr
            self._delete_function_block(start_line, end_line)
        else:
            self._toggle_fold(sig, start_line, end_line)
            self.func_table.selection_set(item)

    def _delete_function_block(self, start_line: int, end_line: int):
        """
        Supprime du code les lignes [start_line..end_line] et,
        si la ligne au-dessus est “-- //////////////////////////////////////////////////////////////////”, la supprime aussi.
        Puis met à jour la table des fonctions et replie tout.
        """
        new_start = start_line
        if start_line >= 2:
            above = self.file_lines[start_line - 2]
            if above.strip() == "-- //////////////////////////////////////////////////////////////////":
                new_start = start_line - 1

        start_idx = f"{new_start}.0"
        end_idx   = f"{end_line+1}.0"
        self.editor_text.delete(start_idx, end_idx)

        content = self.editor_text.get("1.0", tk.END)
        self.file_lines = content.splitlines()
        self._populate_function_table()
        self._fold_all_functions()

    # ──────────────────────────────────────────────────────────────────────────
    # Coloration syntaxique, gutter, auto-indent
    # ──────────────────────────────────────────────────────────────────────────
    def _bind_events(self):
        self.editor_text.bind('<KeyRelease>', lambda e: self._highlight_editor_syntax())
        self.editor_text.bind('<<Modified>>', self._on_editor_modified)
        self.editor_text.bind('<Return>', self._auto_indent)

    def _configure_tags(self):
        # Configurer les tags de coloration dans self.editor_text
        self.editor_text.tag_configure("keyword", foreground=self.theme['keyword'], font=("Consolas", 10, "bold"))
        self.editor_text.tag_configure("comment", foreground=self.theme['comment'], font=("Consolas", 10, "italic"))
        self.editor_text.tag_configure("string",  foreground=self.theme['string'],  font=("Consolas", 10))
        self.editor_text.tag_configure("number",  foreground=self.theme['number'],  font=("Consolas", 10))
        self.editor_text.tag_configure("shaiya",  foreground=self.theme['shaiya'],  font=("Consolas", 10, "bold"))

    def _on_editor_modified(self, event=None):
        if self.editor_text.edit_modified():
            self._highlight_editor_syntax()

    def _editor_on_scroll(self, *args):
        self.editor_text.yview(*args)
        self.editor_gutter.yview(*args)

    def _editor_on_yscroll(self, *args):
        self.editor_text.yview_moveto(args[0])
        self.editor_gutter.yview_moveto(args[0])

    def _auto_indent(self, event=None):
        line, _ = map(int, self.editor_text.index("insert").split('.'))
        curr = self.editor_text.get(f"{line}.0", f"{line}.end")
        indent = re.match(r'^[ \t]*', curr).group(0) if curr else ""
        self.editor_text.insert("insert", "\n" + indent)
        return "break"

    def _update_editor_gutter(self):
        """
        Met à jour la zone des numéros de ligne (gutter) en fonction
        du contenu actuel de l’éditeur.
        """
        self.editor_gutter.config(state='normal')
        self.editor_gutter.delete("1.0", tk.END)
        total = int(self.editor_text.index('end-1c').split('.')[0])
        nums = "".join(f"{i}\n" for i in range(1, total + 1))
        self.editor_gutter.insert("1.0", nums)
        self.editor_gutter.config(state='disabled')

    def _highlight_editor_syntax(self):
        """
        Applique la coloration syntaxique sur tout le contenu de l’éditeur,
        selon les patterns définis.
        """
        content = self.editor_text.get("1.0", tk.END)
        for tag in ("keyword", "comment", "string", "number", "shaiya"):
            self.editor_text.tag_remove(tag, "1.0", tk.END)

        for m in SyntaxPatterns.PATTERN_COMMENT.finditer(content):
            start = f"1.0+{m.start()}c"
            end   = f"1.0+{m.end()}c"
            self.editor_text.tag_add("comment", start, end)

        for m in SyntaxPatterns.PATTERN_STRING.finditer(content):
            idx = m.start()
            if "comment" in self.editor_text.tag_names(f"1.0+{idx}c"):
                continue
            start = f"1.0+{m.start()}c"
            end   = f"1.0+{m.end()}c"
            self.editor_text.tag_add("string", start, end)

        for m in SyntaxPatterns.PATTERN_NUMBER.finditer(content):
            idx = m.start()
            if any(tag in self.editor_text.tag_names(f"1.0+{idx}c") for tag in ("comment", "string")):
                continue
            start = f"1.0+{m.start()}c"
            end   = f"1.0+{m.end()}c"
            self.editor_text.tag_add("number", start, end)

        for m in SyntaxPatterns.PATTERN_KEYWORD.finditer(content):
            idx = m.start()
            if any(tag in self.editor_text.tag_names(f"1.0+{idx}c") for tag in ("comment", "string")):
                continue
            start = f"1.0+{m.start()}c"
            end   = f"1.0+{m.end()}c"
            self.editor_text.tag_add("keyword", start, end)

        for m in SyntaxPatterns.PATTERN_SHAIYA.finditer(content):
            idx = m.start()
            if any(tag in self.editor_text.tag_names(f"1.0+{idx}c") for tag in ("comment", "string")):
                continue
            start = f"1.0+{m.start()}c"
            end   = f"1.0+{m.end()}c"
            self.editor_text.tag_add("shaiya", start, end)

        self._update_editor_gutter()
        self.editor_text.edit_modified(False)

    # ──────────────────────────────────────────────────────────────────────────
    # Insertion de fonctions Shaiya
    # ──────────────────────────────────────────────────────────────────────────
    def _on_shaiya_func_click(self, event):
        """
        Insère un squelette de fonction Shaiya dans l’éditeur, précédé de "-- //////////////////////////////////////////////////////////////////".
        """
        sel = self.func_listbox.curselection()
        if not sel:
            return
        signature = self.func_listbox.get(sel[0])
        snippet = (
            "-- //////////////////////////////////////////////////////////////////\n"
            f"function {signature}\n"
            "    -- TODO : implémenter\n"
            "end\n\n"
        )
        self.editor_text.insert("insert", snippet)
        self._highlight_editor_syntax()
        self._update_editor_gutter()
        content = self.editor_text.get("1.0", tk.END)
        self.file_lines = content.splitlines()
        self._populate_function_table()
        self._fold_all_functions()
        self.func_listbox.selection_clear(0, tk.END)

    # ──────────────────────────────────────────────────────────────────────────
    # Synchronisation : cliquer dans l’éditeur ↔ sélectionner fonction dans le tableau
    # ──────────────────────────────────────────────────────────────────────────
    def _on_editor_click(self, event):
        """
        Quand on clique dans l’éditeur, on détermine la ligne, puis on sélectionne
        la fonction correspondante dans le tableau (si elle existe).
        """
        index = self.editor_text.index("@%d,%d" % (event.x, event.y))
        line = int(index.split(".")[0])
        for key, (start, end, _) in self.functions.items():
            if start <= line <= end:
                # Trouver l’item du tableau qui correspond à `key`
                for item in self.func_table.get_children():
                    sig, s_str, e_str, _ = self.func_table.item(item, "values")
                    if f"{sig}_{s_str}_{e_str}" == key:
                        self.func_table.selection_set(item)
                        self.func_table.see(item)
                        return
        # Aucune fonction → désélection
        self.func_table.selection_remove(self.func_table.selection())

    # ──────────────────────────────────────────────────────────────────────────
    # Contenu du panneau Description, statique
    # ──────────────────────────────────────────────────────────────────────────
    def _get_description_content(self) -> str:
        return (
            "1) Mob = LuaMob(CMob)\n"
            "   - Instancie l’IA serveur du mob à partir de la classe CMob.\n"
            "\n"
            "2) math.randomseed(os.time())\n"
            "   - Initialise la graine aléatoire pour math.random().\n"
            "\n"
            "3) Variables d’état globales :\n"
            "   a) bMobSay = 0       • Indique si le mob a déjà parlé.\n"
            "   b) bMobLoop = 0      • Compteur de boucle IA.\n"
            "   c) bMove = 0         • Indique si le mob se déplace.\n"
            "   d) dwNextSayTime = 0 • Timestamp pour temporisation des dialogues.\n"
            "\n"
            "4) Init()\n"
            "   • Appelée une seule fois à la création du mob (initialisation PV, position, buffs…).\n"
            "\n"
            "5) OnAttacked(dwTime, dwCharID)\n"
            "   • Déclenchée dès qu’un joueur attaque le mob.\n"
            "   • Peut lancer un dialogue si bMobSay = 0.\n"
            "\n"
            "6) WhileCombat(dwTime, dwHPPercent, dwAttackedCount)\n"
            "   • Boucle IA principale en combat.\n"
            "   • Change de phase selon % de vie (80, 60, 40, 20 %).\n"
            "   • Utilise bMobLoop pour ne pas répéter la même phase.\n"
            "   • Peut déclencher un dialogue si os.time() ≥ dwNextSayTime.\n"
            "\n"
            "7) OnDeath(dwTime, dwAttackedCount)\n"
            "   • À la mort du mob : actions (ouverture portail, loot…).\n"
            "\n"
            "8) OnReturnHome(dwTime, dwAttackedCount)\n"
            "   • Mob revient à son point de spawn (hors combat).\n"
            "   • Réinitialise bMobSay, bMobLoop, bMove, dwNextSayTime.\n"
            "\n"
            "9) OnNormalReset(dwTime)\n"
            "   • Réinitialise complètement l’état du mob (après wipe).\n"
            "   • Appelle Mob:LuaReset() et remet tous les drapeaux à 0.\n"
            "\n"
            "10) OnAttackable(dwTime, dwCharID)\n"
            "    • Quand le mob devient attaquable (après cinématique).\n"
            "    • Peut déclencher une animation/effet visuel.\n"
            "\n"
            "11) OnMoveEnd(dwTime)\n"
            "    • À la fin d’un ordre de déplacement programmé.\n"
            "    • Met bMove = 0 pour autoriser un nouveau mouvement.\n"
            "\n"
            "12) Ajouter Fonction (Shaiya)\n"
            "    • Cliquez sur une fonction dans la liste pour insérer son squelette,\n"
            "      précédé de “-- //////////////////////////////////////////////////////////////////”.\n"
            "\n"
            "13) Suppression de fonction\n"
            "    • Cliquez sur la croix “ðﾟﾗﾙ” à droite de la ligne de la fonction dans le tableau\n"
            "      pour supprimer le bloc de code, y compris la ligne “-- //////////////////////////////////////////////////////////////////” si présente.\n"
            "\n"
            "14) Gestion de fichiers\n"
            "    • Charger Dossier : sélectionne un dossier contenant .lua.\n"
            "    • Enregistrer    : sauvegarde le fichier ouvert.\n"
            "    • Nouveau Lua    : crée untitledX.lua automatiquement.\n"
            "    • Supprimer Lua  : supprime le .lua sélectionné du disque.\n"
            "\n"
            "→ Bonnes pratiques :\n"
            "- Toujours fermer chaque bloc `if … then` par `end`, puis chaque fonction par `end`.\n"
            "- Initialiser toutes les variables globales ou locales avant utilisation.\n"
            "- Préférez `local` pour les variables non globales.\n"
            "- Gérer chaque état IA (combat, dialogues, déplacements) avec des drapeaux appropriés.\n"
        )
# ──────────────────────────────────────────────────────────────────────────────
# MAP.INI EDITOR (onglet "EditMapCreate")
# ──────────────────────────────────────────────────────────────────────────────

class MapIniEditorFrame(tk.Frame):
    """
    Frame qui contient l’éditeur de fichier map.ini dans l’onglet EditMapCreate.
    """
    def __init__(self, parent):
        super().__init__(parent)
        # Appliquer style sombre ttk
        self.apply_dark_theme()

        # Dictionnaires de textes (français uniquement)
        self.text = {
            'title': 'Éditeur map.ini - Mode Sombre',
            'menu_open_btn': 'ðﾟﾓﾂ Charger Fichier',
            'menu_save_all_btn': 'ðﾟﾒﾾ Sauvegarder Fichier',
            'error_no_file': 'Aucun fichier sélectionné.',
            'sections': 'Sections :',
            'add_section_btn': '➕ Ajouter Section',
            'delete_section_btn': '❌ Supprimer Section',
            'menu_options': "Menu d'options :",
            'values': ['Affichage Général', 'Réglages Carte', 'Réglages Météo'],
            'general_hint': 'Double-cliquez pour éditer.',
            'maptype': 'MapType :',
            'mapname': 'MapName :',
            'rebirth': 'RebirthMapPos :',
            'rebirth1': 'RebirthMapPos1 :',
            'rebirth2': 'RebirthMapPos2 :',
            'mapcurse': 'MapCurse :',
            'wartype': 'WarType :',
            'createtype': 'CreateType :',
            'createmin': 'CreateMinUser :',
            'entermax': 'EnterMaxUser :',
            'expiretime': 'ExpireTime :',
            'gkmin': 'GateKeeperLvMin :',
            'gkmax': 'GateKeeperLvMax :',
            'save_map': 'Enregistrer Carte',
            'desc_map': (
                'MapType : Type de la zone.\n'
                '  F : Champ\n'
                '  C : Cité\n'
                '  D : Donjon\n'
                '  G : Ville\n'
                'MapName : ID numérique ou nom affiché. -1 pour aucun.\n'
                'RebirthMapPos : Réapparition par défaut. Forme : map_id, x, y, z.\n'
                '  Ex : 70, 155.23, 38.42, 175.27\n'
                'RebirthMapPos1 : Réapparition équipe A. Forme : map_id, x, y, z.\n'
                '  Ex : 70, 763, 28, 1258\n'
                'RebirthMapPos2 : Réapparition équipe B. Forme : map_id, x, y, z.\n'
                '  Ex : 70, 1319, 49, 1472\n'
                'MapCurse : Paramètres de malédiction. Forme : actif(0/1), id_mal, param0, param1.\n'
                '  Ex : 1, 372, 0, 1\n'
                'WarType : Mode JcJ. P = JcJ activé.\n'
                'CreateType : Type d’instance. P = Partie.\n'
                'CreateMinUser : Nombre min de joueurs pour créer.\n'
                'EnterMaxUser : Nombre max de joueurs pour entrer.\n'
                'ExpireTime : Minutes avant expiration de l’instance.\n'
                'GateKeeperLvMin : Niveau min (ex : 10).\n'
                'GateKeeperLvMax : Niveau max (ex : 15).\n'
            ),
            'maptype_values': ['F : Champ', 'C : Cité', 'D : Donjon', 'G : Ville'],
            'createtype_values': ['P : Partie', 'A : Aléatoire', 'G : Guilde'],
            'wartype_values': ['P : JcJ', 'N : Aucun'],
            'weatherstate': 'WeatherState :',
            'weatherrate': 'WeatherRate :',
            'weatherpower': 'WeatherPower :',
            'weatherdelay': 'WeatherDelay :',
            'weathernonedelay': 'WeatherNoneDelay :',
            'save_weather': 'Enregistrer Météo',
            'desc_weather': (
                'WeatherState :\n'
                '  0 : Clair\n'
                '  1 : Pluie légère\n'
                '  2 : Pluie\n'
                '  3 : Neige\n'
                'WeatherRate : Pourcentage (0-100%) d’apparition.\n'
                'WeatherPower : Intensité (plus élevé = plus fort).\n'
                'WeatherDelay : Secondes entre rafales.\n'
                'WeatherNoneDelay : Secondes sans météo.\n'
            ),
            'weatherstate_values': ['0 : Clair', '1 : Pluie légère', '2 : Pluie', '3 : Neige'],
            'info_map_saved': 'Réglages carte enregistrés.',
            'info_weather_saved': 'Réglages météo enregistrés.',
            'info_saved_all': 'Fichier enregistré : {}',
            'error_save': 'Impossible d’enregistrer : {}',
            'confirm_delete': "Supprimer '{}' ?"
        }

        # Initialisation
        self.CONFIG_FILE = None
        self.config_parser = configparser.ConfigParser(
            interpolation=None, delimiters=['='], comment_prefixes=['//'], strict=False, allow_no_value=True
        )
        self.config_parser.optionxform = str
        self.current_section = None

        # Construire les widgets
        self.create_widgets()

    def apply_dark_theme(self):
        """
        Applique le style sombre aux widgets Ttk.
        """
        style = ttk.Style(self)
        style.theme_use('clam')
        dark_bg = '#2e2e2e'
        dark_fg = '#ffffff'
        field_bg = '#3e3e3e'
        select_bg = '#5a5a5a'

        style.configure('.', background=dark_bg, foreground=dark_fg)
        style.configure('TLabel', background=dark_bg, foreground=dark_fg)
        style.configure('TFrame', background=dark_bg)
        style.configure('TButton', background=field_bg, foreground=dark_fg)
        style.map('TButton', background=[('active', select_bg)])
        style.configure('TCombobox', fieldbackground=field_bg, background=field_bg, foreground=dark_fg)
        style.configure('TEntry', fieldbackground=field_bg, background=field_bg, foreground=dark_fg)
        style.configure('Treeview', background=field_bg, foreground=dark_fg, fieldbackground=field_bg)
        style.map('Treeview', background=[('selected', select_bg)], foreground=[('selected', dark_fg)])

        # Couleurs pour Listbox (non-Ttk)
        self.listbox_bg = field_bg
        self.listbox_fg = dark_fg
        self.listbox_select_bg = select_bg
        self.listbox_select_fg = dark_fg

    def load_or_create_config(self):
        if not self.CONFIG_FILE:
            return

        # Si le fichier n'existe pas, on le crée vide
        if self.CONFIG_FILE and not os.path.exists(self.CONFIG_FILE):
            open(self.CONFIG_FILE, 'w', encoding='utf-8').close()

        # Lire le contenu brut
        with open(self.CONFIG_FILE, 'r', encoding='utf-8') as f:
            raw = f.read().splitlines()

        # Assurer qu'il y a au moins une section en tête
        i = 0
        while i < len(raw) and raw[i].strip() == '':
            i += 1
        if i < len(raw):
            first = raw[i].lstrip()
            if not first.startswith('['):
                raw.insert(i, '[__DEFAULT__]')

        text_to_read = "\n".join(raw)
        try:
            self.config_parser.read_string(text_to_read)
        except configparser.MissingSectionHeaderError:
            # En cas d'erreur, on initialise un parser minimal
            self.config_parser = configparser.ConfigParser(
                interpolation=None, delimiters=['='], comment_prefixes=['//'], strict=False, allow_no_value=True
            )
            self.config_parser.optionxform = str
            self.config_parser.read_string("[__DEFAULT__]\n")

    def create_widgets(self):
        paned = ttk.Panedwindow(self, orient=tk.HORIZONTAL)
        paned.pack(fill=tk.BOTH, expand=True)

        # Colonne gauche : liste des sections + boutons
        self.left_frame = ttk.Frame(paned, width=250)
        paned.add(self.left_frame, weight=1)

        self.sections_label = ttk.Label(
            self.left_frame, text=self.text['sections'], font=('Arial', 10, 'bold')
        )
        self.sections_label.pack(anchor=tk.W, padx=5, pady=(5, 0))

        self.section_listbox = tk.Listbox(
            self.left_frame,
            bg=self.listbox_bg,
            fg=self.listbox_fg,
            selectbackground=self.listbox_select_bg,
            selectforeground=self.listbox_select_fg,
            exportselection=False,
            height=20
        )
        self.section_listbox.pack(fill=tk.BOTH, expand=True, padx=5, pady=(0, 5))
        self.section_listbox.bind('<<ListboxSelect>>', self.on_section_select)

        btns_frame = ttk.Frame(self.left_frame)
        btns_frame.pack(fill=tk.X, padx=5, pady=(0, 10))

        self.btn_add = ttk.Button(btns_frame, text=self.text['add_section_btn'], command=self.add_section)
        self.btn_add.pack(fill=tk.X, pady=2)
        self.btn_delete = ttk.Button(btns_frame, text=self.text['delete_section_btn'], command=self.delete_section)
        self.btn_delete.pack(fill=tk.X, pady=2)

        ttk.Separator(self.left_frame, orient=tk.HORIZONTAL).pack(fill=tk.X, padx=5, pady=(10, 5))

        self.btn_open = ttk.Button(self.left_frame, text=self.text['menu_open_btn'], command=self.open_file)
        self.btn_open.pack(fill=tk.X, padx=5, pady=2)
        self.btn_save_all = ttk.Button(self.left_frame, text=self.text['menu_save_all_btn'], command=self.save_file)
        self.btn_save_all.pack(fill=tk.X, padx=5, pady=(2, 10))

        # Colonne droite : menu options + contenu
        center_frame = ttk.Frame(paned)
        paned.add(center_frame, weight=2)

        self.menu_label = ttk.Label(center_frame, text=self.text['menu_options'], font=('Arial', 10, 'bold'))
        self.menu_label.pack(anchor=tk.W, padx=5, pady=(5, 0))
        self.option_var = tk.StringVar()
        self.option_menu = ttk.Combobox(center_frame, textvariable=self.option_var, state='readonly')
        self.option_menu['values'] = self.text['values']
        self.option_menu.current(0)
        self.option_menu.pack(fill=tk.X, padx=5, pady=5)
        self.option_menu.bind('<<ComboboxSelected>>', self.on_option_change)

        self.content_frame = ttk.Frame(center_frame)
        self.content_frame.pack(fill=tk.BOTH, expand=True, padx=5, pady=5)
        self.frames = {}
        for mode in ['general', 'map', 'weather']:
            frame = ttk.Frame(self.content_frame)
            self.frames[mode] = frame
            frame.grid(row=0, column=0, sticky='nsew')

        # —— Vue générale —— #
        cols = ('Key', 'Value')
        self.general_tree = ttk.Treeview(
            self.frames['general'], columns=cols, show='headings', style='Treeview'
        )
        for c in cols:
            self.general_tree.heading(c, text=c)
            self.general_tree.column(c, width=200, anchor=tk.W)
        self.general_tree.pack(fill=tk.BOTH, expand=True)
        self.general_hint = ttk.Label(
            self.frames['general'],
            text=self.text['general_hint'],
            font=('Arial', 8, 'italic'),
            background=self.listbox_bg,
            foreground=self.listbox_fg
        )
        self.general_hint.pack(anchor=tk.W, pady=(2, 0))
        self.general_tree.bind('<Double-1>', self.edit_general_cell)

        # —— Réglages carte —— #
        self.map_vars = {
            'MapType': tk.StringVar(),
            'MapName': tk.StringVar(),
            'RebirthMapPos': tk.StringVar(),
            'RebirthMapPos1': tk.StringVar(),
            'RebirthMapPos2': tk.StringVar(),
            'MapCurse': tk.StringVar(),
            'WarType': tk.StringVar(),
            'CreateType': tk.StringVar(),
            'CreateMinUser': tk.StringVar(),
            'EnterMaxUser': tk.StringVar(),
            'ExpireTime': tk.StringVar(),
            'GateKeeperLvMin': tk.StringVar(),
            'GateKeeperLvMax': tk.StringVar()
        }
        map_frame = self.frames['map']

        self.map_label_type = ttk.Label(map_frame, text=self.text['maptype'])
        self.map_label_type.grid(row=0, column=0, sticky=tk.W, pady=5)
        self.map_type_combo = ttk.Combobox(
            map_frame, textvariable=self.map_vars['MapType'], state='readonly'
        )
        self.map_type_combo['values'] = self.text['maptype_values']
        self.map_type_combo.grid(row=0, column=1, sticky=tk.EW, pady=5)

        self.map_label_name = ttk.Label(map_frame, text=self.text['mapname'])
        self.map_label_name.grid(row=1, column=0, sticky=tk.W, pady=5)
        self.map_name_entry = ttk.Entry(map_frame, textvariable=self.map_vars['MapName'])
        self.map_name_entry.grid(row=1, column=1, sticky=tk.EW, pady=5)

        self.map_label_rebirth = ttk.Label(map_frame, text=self.text['rebirth'])
        self.map_label_rebirth.grid(row=2, column=0, sticky=tk.W, pady=5)
        self.rebirth_entry = ttk.Entry(map_frame, textvariable=self.map_vars['RebirthMapPos'])
        self.rebirth_entry.grid(row=2, column=1, sticky=tk.EW, pady=5)

        self.map_label_rebirth1 = ttk.Label(map_frame, text=self.text['rebirth1'])
        self.map_label_rebirth1.grid(row=3, column=0, sticky=tk.W, pady=5)
        self.rebirth1_entry = ttk.Entry(map_frame, textvariable=self.map_vars['RebirthMapPos1'])
        self.rebirth1_entry.grid(row=3, column=1, sticky=tk.EW, pady=5)

        self.map_label_rebirth2 = ttk.Label(map_frame, text=self.text['rebirth2'])
        self.map_label_rebirth2.grid(row=4, column=0, sticky=tk.W, pady=5)
        self.rebirth2_entry = ttk.Entry(map_frame, textvariable=self.map_vars['RebirthMapPos2'])
        self.rebirth2_entry.grid(row=4, column=1, sticky=tk.EW, pady=5)

        self.map_label_curse = ttk.Label(map_frame, text=self.text['mapcurse'])
        self.map_label_curse.grid(row=5, column=0, sticky=tk.W, pady=5)
        self.map_curse_entry = ttk.Entry(map_frame, textvariable=self.map_vars['MapCurse'])
        self.map_curse_entry.grid(row=5, column=1, sticky=tk.EW, pady=5)

        self.map_label_warType = ttk.Label(map_frame, text=self.text['wartype'])
        self.map_label_warType.grid(row=6, column=0, sticky=tk.W, pady=5)
        self.war_type_combo = ttk.Combobox(
            map_frame, textvariable=self.map_vars['WarType'], state='readonly'
        )
        self.war_type_combo['values'] = self.text['wartype_values']
        self.war_type_combo.grid(row=6, column=1, sticky=tk.EW, pady=5)

        self.map_label_createtype = ttk.Label(map_frame, text=self.text['createtype'])
        self.map_label_createtype.grid(row=7, column=0, sticky=tk.W, pady=5)
        self.create_type_combo = ttk.Combobox(
            map_frame, textvariable=self.map_vars['CreateType'], state='readonly'
        )
        self.create_type_combo['values'] = self.text['createtype_values']
        self.create_type_combo.grid(row=7, column=1, sticky=tk.EW, pady=5)

        self.map_label_createmin = ttk.Label(map_frame, text=self.text['createmin'])
        self.map_label_createmin.grid(row=8, column=0, sticky=tk.W, pady=5)
        self.create_min_entry = ttk.Entry(map_frame, textvariable=self.map_vars['CreateMinUser'])
        self.create_min_entry.grid(row=8, column=1, sticky=tk.EW, pady=5)

        self.map_label_entermax = ttk.Label(map_frame, text=self.text['entermax'])
        self.map_label_entermax.grid(row=9, column=0, sticky=tk.W, pady=5)
        self.enter_max_entry = ttk.Entry(map_frame, textvariable=self.map_vars['EnterMaxUser'])
        self.enter_max_entry.grid(row=9, column=1, sticky=tk.EW, pady=5)

        self.map_label_expiretime = ttk.Label(map_frame, text=self.text['expiretime'])
        self.map_label_expiretime.grid(row=10, column=0, sticky=tk.W, pady=5)
        self.expire_time_entry = ttk.Entry(map_frame, textvariable=self.map_vars['ExpireTime'])
        self.expire_time_entry.grid(row=10, column=1, sticky=tk.EW, pady=5)

        self.map_label_gkmin = ttk.Label(map_frame, text=self.text['gkmin'])
        self.map_label_gkmin.grid(row=11, column=0, sticky=tk.W, pady=5)
        self.gkmin_entry = ttk.Entry(map_frame, textvariable=self.map_vars['GateKeeperLvMin'])
        self.gkmin_entry.grid(row=11, column=1, sticky=tk.EW, pady=5)

        self.map_label_gkmax = ttk.Label(map_frame, text=self.text['gkmax'])
        self.map_label_gkmax.grid(row=12, column=0, sticky=tk.W, pady=5)
        self.gkmax_entry = ttk.Entry(map_frame, textvariable=self.map_vars['GateKeeperLvMax'])
        self.gkmax_entry.grid(row=12, column=1, sticky=tk.EW, pady=5)

        self.btn_save_map = ttk.Button(map_frame, text=self.text['save_map'], command=self.save_map_settings)
        self.btn_save_map.grid(row=13, column=0, columnspan=2, pady=10)
        map_frame.columnconfigure(1, weight=1)

        self.desc_map = tk.Text(map_frame, height=16, background=self.listbox_bg, foreground=self.listbox_fg)
        self.desc_map.grid(row=0, column=2, rowspan=14, sticky='nsew', padx=(10, 0))
        map_frame.columnconfigure(2, weight=1)
        self.desc_map.insert(tk.END, self.text['desc_map'])
        self.desc_map.configure(state='disabled')

        # —— Réglages météo —— #
        self.weather_vars = {
            'WeatherState': tk.StringVar(),
            'WeatherRate': tk.StringVar(),
            'WeatherPower': tk.StringVar(),
            'WeatherDelay': tk.StringVar(),
            'WeatherNoneDelay': tk.StringVar()
        }
        weather_frame = self.frames['weather']

        self.weather_label_state = ttk.Label(weather_frame, text=self.text['weatherstate'])
        self.weather_label_state.grid(row=0, column=0, sticky=tk.W, pady=5)
        self.weather_state_combo = ttk.Combobox(
            weather_frame, textvariable=self.weather_vars['WeatherState'], state='readonly'
        )
        self.weather_state_combo['values'] = self.text['weatherstate_values']
        self.weather_state_combo.grid(row=0, column=1, sticky=tk.EW, pady=5)

        self.weather_label_rate = ttk.Label(weather_frame, text=self.text['weatherrate'])
        self.weather_label_rate.grid(row=1, column=0, sticky=tk.W, pady=5)
        self.weather_rate_entry = ttk.Entry(weather_frame, textvariable=self.weather_vars['WeatherRate'])
        self.weather_rate_entry.grid(row=1, column=1, sticky=tk.EW, pady=5)

        self.weather_label_power = ttk.Label(weather_frame, text=self.text['weatherpower'])
        self.weather_label_power.grid(row=2, column=0, sticky=tk.W, pady=5)
        self.weather_power_entry = ttk.Entry(weather_frame, textvariable=self.weather_vars['WeatherPower'])
        self.weather_power_entry.grid(row=2, column=1, sticky=tk.EW, pady=5)

        self.weather_label_delay = ttk.Label(weather_frame, text=self.text['weatherdelay'])
        self.weather_label_delay.grid(row=3, column=0, sticky=tk.W, pady=5)
        self.weather_delay_entry = ttk.Entry(weather_frame, textvariable=self.weather_vars['WeatherDelay'])
        self.weather_delay_entry.grid(row=3, column=1, sticky=tk.EW, pady=5)

        self.weather_label_nonedelay = ttk.Label(weather_frame, text=self.text['weathernonedelay'])
        self.weather_label_nonedelay.grid(row=4, column=0, sticky=tk.W, pady=5)
        self.weather_nonedelay_entry = ttk.Entry(
            weather_frame, textvariable=self.weather_vars['WeatherNoneDelay']
        )
        self.weather_nonedelay_entry.grid(row=4, column=1, sticky=tk.EW, pady=5)

        self.btn_save_weather = ttk.Button(
            weather_frame, text=self.text['save_weather'], command=self.save_weather_settings
        )
        self.btn_save_weather.grid(row=5, column=0, columnspan=2, pady=10)
        weather_frame.columnconfigure(1, weight=1)

        self.desc_weather = tk.Text(weather_frame, height=10, background=self.listbox_bg, foreground=self.listbox_fg)
        self.desc_weather.grid(row=0, column=2, rowspan=6, sticky='nsew', padx=(10, 0))
        weather_frame.columnconfigure(2, weight=1)
        self.desc_weather.insert(tk.END, self.text['desc_weather'])
        self.desc_weather.configure(state='disabled')

        # Afficher la vue Générale au démarrage
        self.show_frame('general')

    def populate_sections(self):
        self.section_listbox.delete(0, tk.END)
        zones = list(self.config_parser.sections())
        for section in zones:
            self.section_listbox.insert(tk.END, section)

        if zones:
            first = zones[0]
            self.current_section = first
            self.section_listbox.select_set(0)
            self.load_section_data()

    def on_section_select(self, event=None):
        sel = self.section_listbox.curselection()
        if not sel:
            return
        idx = sel[0]
        chosen = self.section_listbox.get(idx)
        self.current_section = chosen
        self.load_section_data()

    def load_section_data(self):
        def strip_comment(val):
            return val.split('//')[0].strip()

        if not self.current_section:
            return

        data = self.config_parser[self.current_section]

        # Met à jour le Treeview général
        for row in self.general_tree.get_children():
            self.general_tree.delete(row)
        for key, value in data.items():
            self.general_tree.insert('', tk.END, values=(key, value))

        # MapType
        raw_maptype = strip_comment(data.get('MapType', ''))
        found = False
        for item in self.text['maptype_values']:
            code = item.split(':')[0].strip()
            if code == raw_maptype:
                self.map_vars['MapType'].set(item)
                found = True
                break
        if not found:
            self.map_vars['MapType'].set('')

        # MapName
        self.map_vars['MapName'].set(strip_comment(data.get('MapName', '')))

        # RebirthMapPos
        self.map_vars['RebirthMapPos'].set(strip_comment(data.get('RebirthMapPos', '')))
        self.map_vars['RebirthMapPos1'].set(strip_comment(data.get('RebirthMapPos1', '')))
        self.map_vars['RebirthMapPos2'].set(strip_comment(data.get('RebirthMapPos2', '')))

        # MapCurse
        self.map_vars['MapCurse'].set(strip_comment(data.get('MapCurse', '')))

        # WarType
        raw_war = strip_comment(data.get('WarType', ''))
        found_w = False
        for item in self.text['wartype_values']:
            code = item.split(':')[0].strip()
            if code == raw_war:
                self.map_vars['WarType'].set(item)
                found_w = True
                break
        if not found_w:
            self.map_vars['WarType'].set('')

        # CreateType
        raw_ct = strip_comment(data.get('CreateType', ''))
        found_ct = False
        for item in self.text['createtype_values']:
            code = item.split(':')[0].strip()
            if code == raw_ct:
                self.map_vars['CreateType'].set(item)
                found_ct = True
                break
        if not found_ct:
            self.map_vars['CreateType'].set('')

        # CreateMinUser, EnterMaxUser, ExpireTime
        self.map_vars['CreateMinUser'].set(strip_comment(data.get('CreateMinUser', '')))
        self.map_vars['EnterMaxUser'].set(strip_comment(data.get('EnterMaxUser', '')))
        self.map_vars['ExpireTime'].set(strip_comment(data.get('ExpireTime', '')))

        # GateKeeperLvMin, GateKeeperLvMax
        self.map_vars['GateKeeperLvMin'].set(strip_comment(data.get('GateKeeperLvMin', '')))
        self.map_vars['GateKeeperLvMax'].set(strip_comment(data.get('GateKeeperLvMax', '')))

        # WeatherState
        raw_weatherstate = strip_comment(data.get('WeatherState', ''))
        found_ws = False
        for item in self.text['weatherstate_values']:
            code = item.split(':')[0].strip()
            if code == raw_weatherstate:
                self.weather_vars['WeatherState'].set(item)
                found_ws = True
                break
        if not found_ws:
            self.weather_vars['WeatherState'].set('')

        # WeatherRate, WeatherPower, WeatherDelay, WeatherNoneDelay
        self.weather_vars['WeatherRate'].set(strip_comment(data.get('WeatherRate', '')))
        self.weather_vars['WeatherPower'].set(strip_comment(data.get('WeatherPower', '')))
        self.weather_vars['WeatherDelay'].set(strip_comment(data.get('WeatherDelay', '')))
        self.weather_vars['WeatherNoneDelay'].set(strip_comment(data.get('WeatherNoneDelay', '')))

    def on_option_change(self, event=None):
        choice = self.option_var.get()
        if choice == self.text['values'][0]:
            self.show_frame('general')
        elif choice == self.text['values'][1]:
            self.show_frame('map')
        elif choice == self.text['values'][2]:
            self.show_frame('weather')

    def show_frame(self, mode):
        self.frames[mode].tkraise()

    def edit_general_cell(self, event):
        item = self.general_tree.identify_row(event.y)
        col = self.general_tree.identify_column(event.x)
        if not item or not col:
            return
        key, val = self.general_tree.item(item, 'values')
        nouveau = simpledialog.askstring(
            self.text['values'][0],
            f"Nouvelle valeur pour '{key}' :",
            initialvalue=val
        )
        if nouveau is not None:
            self.general_tree.set(item, 'Value', nouveau)

    def save_general(self):
        sec = self.config_parser[self.current_section]
        for existing_key in list(sec.keys()):
            sec.pop(existing_key, None)
        for row_id in self.general_tree.get_children():
            key, val = self.general_tree.item(row_id, 'values')
            if key.strip() and val.strip():
                sec[key] = val

    def save_map_settings(self):
        if not self.current_section:
            return
        sec = self.config_parser[self.current_section]
        # MapType
        sel = self.map_vars['MapType'].get()
        code = sel.split(':')[0].strip() if sel else ''
        if code:
            sec['MapType'] = code
        elif 'MapType' in sec:
            del sec['MapType']
        # MapName
        name = self.map_vars['MapName'].get().strip()
        if name:
            sec['MapName'] = name
        elif 'MapName' in sec:
            del sec['MapName']
        # RebirthMapPos / RebirthMapPos1 / RebirthMapPos2
        for key, var_name in [
            ('RebirthMapPos', 'RebirthMapPos'),
            ('RebirthMapPos1', 'RebirthMapPos1'),
            ('RebirthMapPos2', 'RebirthMapPos2')
        ]:
            val = self.map_vars[var_name].get().strip()
            if val:
                sec[key] = val
            elif key in sec:
                del sec[key]
        # MapCurse
        curse = self.map_vars['MapCurse'].get().strip()
        if curse:
            sec['MapCurse'] = curse
        elif 'MapCurse' in sec:
            del sec['MapCurse']
        # WarType
        wsel = self.map_vars['WarType'].get()
        wcode = wsel.split(':')[0].strip() if wsel else ''
        if wcode:
            sec['WarType'] = wcode
        elif 'WarType' in sec:
            del sec['WarType']
        # CreateType
        ct_sel = self.map_vars['CreateType'].get()
        ct_code = ct_sel.split(':')[0].strip() if ct_sel else ''
        if ct_code:
            sec['CreateType'] = ct_code
        elif 'CreateType' in sec:
            del sec['CreateType']
        # CreateMinUser / EnterMaxUser / ExpireTime
        for key, var_name in [
            ('CreateMinUser', 'CreateMinUser'),
            ('EnterMaxUser', 'EnterMaxUser'),
            ('ExpireTime', 'ExpireTime')
        ]:
            val = self.map_vars[var_name].get().strip()
            if val:
                sec[key] = val
            elif key in sec:
                del sec[key]
        # GateKeeperLvMin / GateKeeperLvMax
        for key, var_name in [
            ('GateKeeperLvMin', 'GateKeeperLvMin'),
            ('GateKeeperLvMax', 'GateKeeperLvMax')
        ]:
            val = self.map_vars[var_name].get().strip()
            if val:
                sec[key] = val
            elif key in sec:
                del sec[key]

        messagebox.showinfo('', self.text['info_map_saved'])
        self.load_section_data()

    def save_weather_settings(self):
        if not self.current_section:
            return
        sec = self.config_parser[self.current_section]
        # WeatherState
        sel = self.weather_vars['WeatherState'].get()
        code = sel.split(':')[0].strip() if sel else ''
        if code:
            sec['WeatherState'] = code
        elif 'WeatherState' in sec:
            del sec['WeatherState']
        # WeatherRate / WeatherPower / WeatherDelay / WeatherNoneDelay
        for key, var_name in [
            ('WeatherRate', 'WeatherRate'),
            ('WeatherPower', 'WeatherPower'),
            ('WeatherDelay', 'WeatherDelay'),
            ('WeatherNoneDelay', 'WeatherNoneDelay')
        ]:
            val = self.weather_vars[var_name].get().strip()
            if val:
                sec[key] = val
            elif key in sec:
                del sec[key]

        messagebox.showinfo('', self.text['info_weather_saved'])
        self.load_section_data()

    def save_file(self):
        """
        Sauvegarde l'ensemble du fichier : on met à jour d'abord la section courante,
        puis on écrit tout le fichier sur le disque.
        """
        if not self.CONFIG_FILE or not self.current_section:
            messagebox.showwarning('', self.text['error_no_file'])
            return

        self.save_general()
        self.save_map_settings()
        self.save_weather_settings()
        try:
            with open(self.CONFIG_FILE, 'w', encoding='utf-8') as f:
                self.config_parser.write(f, space_around_delimiters=True)
            messagebox.showinfo('', self.text['info_saved_all'].format(self.CONFIG_FILE))
        except Exception as e:
            messagebox.showerror('', self.text['error_save'].format(e))

    def open_file(self):
        path = filedialog.askopenfilename(
            title=self.text['menu_open_btn'],
            filetypes=[('INI', '*.ini')]
        )
        if not path:
            return
        self.CONFIG_FILE = path
        # Recharger la config
        self.load_or_create_config()
        # Mettre à jour la liste des sections
        self.populate_sections()

    def add_section(self):
        if not self.CONFIG_FILE:
            messagebox.showwarning('', self.text['error_no_file'])
            return

        max_index = -1
        for name in self.config_parser.sections():
            if name.startswith('SET_ZONE_'):
                parts = name.split('_')
                try:
                    idx = int(parts[-1])
                    if idx > max_index:
                        max_index = idx
                except ValueError:
                    pass
        next_index = max_index + 1
        new_name = f'SET_ZONE_{next_index}'
        if 'DEFAULT_SET_ZONE' in self.config_parser:
            default = self.config_parser['DEFAULT_SET_ZONE']
            self.config_parser[new_name] = {}
            for key, val in default.items():
                self.config_parser[new_name][key] = val
        else:
            self.config_parser[new_name] = {}

        self.populate_sections()
        idx = list(self.config_parser.sections()).index(new_name)
        self.section_listbox.select_clear(0, tk.END)
        self.section_listbox.select_set(idx)
        self.current_section = new_name
        self.load_section_data()

    def delete_section(self):
        sel = self.section_listbox.curselection()
        if not sel:
            return
        idx = sel[0]
        section = self.section_listbox.get(idx)
        if messagebox.askyesno('', self.text['confirm_delete'].format(section)):
            self.config_parser.remove_section(section)
            self.populate_sections()
# ──────────────────────────────────────────────────────────────────────────────
# ChaoticSquare EDITOR (onglet "Edit")
# ──────────────────────────────────────────────────────────────────────────────
# -*- coding: utf-8 -*-
import tkinter as tk
from tkinter import filedialog, messagebox
from collections import OrderedDict

# Texte en français uniquement
texts = {
    'title': "EditChaoticSquare",
    'load_file': "ðﾟﾓﾂ Charger Fichier",
    'save_file': "ðﾟﾒﾾ Sauvegarder Fichier",
    'add_section': "➕ Ajouter Section",
    'delete_section': "❌ Supprimer Section",
    'sections': "Sections",
    'status_no_file': "Aucun fichier chargé.",
    'confirm_delete': "Confirmation suppression",
    'confirm_delete_msg': "Voulez-vous vraiment supprimer la section « {section} » ?",
    'deleted_msg': "La section a bien été supprimée.",
    'saved_msg': "Le fichier a bien été mis à jour.",
    'warning_no_file': "Aucun fichier",
    'warning_no_file_msg': "Aucun fichier n'a été chargé.",
    'warning_no_section': "Aucune section",
    'warning_no_section_msg': "Veuillez sélectionner une section à modifier.",
}

class ChaoticSquareEditorFrame(tk.Frame):
    """
    Anciennement ItemSynthesisEditor, transformé pour être un Frame
    embarqué dans un Notebook (onglet "EditChaoticSquare").
    """
    def __init__(self, parent):
        super().__init__(parent, bg='#2e2e2e')
        self.parent = parent

        # Couleurs en mode sombre
        self.bg_color = '#2e2e2e'
        self.fg_color = '#ffffff'
        self.entry_bg = '#3e3e3e'
        self.button_bg = '#444444'
        self.configure(bg=self.bg_color)

        # Données internes
        self.data = OrderedDict()
        self.current_file = None
        self.current_section = None

        # --- Barre de commandes (boutons) ---
        top_frame = tk.Frame(self, bg=self.bg_color)
        top_frame.pack(fill=tk.X, padx=5, pady=5)

        self.btn_load = tk.Button(
            top_frame,
            text=texts['load_file'],
            command=self.load_file_dialog,
            bg=self.button_bg, fg=self.fg_color, activebackground=self.entry_bg
        )
        self.btn_load.pack(side=tk.LEFT, padx=5)

        self.btn_save = tk.Button(
            top_frame,
            text=texts['save_file'],
            command=self.on_save_changes,
            bg=self.button_bg, fg=self.fg_color, activebackground=self.entry_bg
        )
        self.btn_save.pack(side=tk.LEFT, padx=5)

        self.btn_add = tk.Button(
            top_frame,
            text=texts['add_section'],
            command=self.on_add_section,
            bg=self.button_bg, fg=self.fg_color, activebackground=self.entry_bg
        )
        self.btn_add.pack(side=tk.LEFT, padx=20)

        self.btn_delete = tk.Button(
            top_frame,
            text=texts['delete_section'],
            command=self.on_delete_section,
            bg=self.button_bg, fg=self.fg_color, activebackground=self.entry_bg
        )
        self.btn_delete.pack(side=tk.LEFT, padx=5)

        # --- Cadre principal : liste des sections à gauche, détails à droite ---
        main_frame = tk.Frame(self, bg=self.bg_color)
        main_frame.pack(fill=tk.BOTH, expand=True, padx=5, pady=5)

        # -- Liste des sections --
        list_frame = tk.Frame(main_frame, bg=self.bg_color)
        list_frame.pack(side=tk.LEFT, fill=tk.Y)

        self.lbl_sections = tk.Label(
            list_frame,
            text=texts['sections'],
            bg=self.bg_color, fg=self.fg_color
        )
        self.lbl_sections.pack(anchor=tk.NW)

        self.listbox = tk.Listbox(
            list_frame,
            width=30,
            activestyle="none",
            bg=self.entry_bg,
            fg=self.fg_color,
            selectbackground='#555555',
            selectforeground=self.fg_color
        )
        self.listbox.pack(fill=tk.Y, expand=True)
        self.listbox.bind("<<ListboxSelect>>", self.on_select_section)

        # -- Détails de la section sélectionnée --
        detail_container = tk.Frame(main_frame, bg=self.bg_color)
        detail_container.pack(side=tk.RIGHT, fill=tk.BOTH, expand=True, padx=(10,0), pady=5)

        # Canvas + scrollbar pour faire défiler si nécessaire
        self.canvas = tk.Canvas(detail_container, bg=self.bg_color, highlightthickness=0)
        self.canvas.pack(side=tk.LEFT, fill=tk.BOTH, expand=True)

        scrollbar = tk.Scrollbar(detail_container, orient=tk.VERTICAL, command=self.canvas.yview)
        scrollbar.pack(side=tk.RIGHT, fill=tk.Y)
        self.canvas.configure(yscrollcommand=scrollbar.set)

        self.detail_frame = tk.Frame(self.canvas, bg=self.bg_color)
        self.canvas.create_window((0,0), window=self.detail_frame, anchor='nw')
        self.detail_frame.bind(
            "<Configure>",
            lambda e: self.canvas.configure(scrollregion=self.canvas.bbox("all"))
        )

        # Champs (clés) dans l’ordre d’affichage
        self.fields = [
            "ItemID", "SuccessRate",
            "MaterialType", "MaterialTypeID", "MaterialCount",
            "CreateType", "CreateTypeID", "CreateCount"
        ]
        self.entries = {}       # dict[field] = Entry widget (pour champs simples)
        self.list_entries = {}  # dict[field] = liste de 24 Entry widgets (pour champs liste)

        starting_row = 0
        for field in self.fields:
            lbl = tk.Label(
                self.detail_frame,
                text=field,
                bg=self.bg_color,
                fg=self.fg_color
            )
            lbl.grid(row=starting_row, column=0, sticky=tk.W, pady=(5,0))

            if field in ("MaterialType", "MaterialTypeID", "MaterialCount"):
                subframe = tk.Frame(self.detail_frame, bg=self.bg_color)
                subframe.grid(row=starting_row + 1, column=0, sticky=tk.W)

                entries = []
                for i in range(24):
                    e = tk.Entry(
                        subframe,
                        width=3,
                        bg=self.entry_bg,
                        fg=self.fg_color,
                        insertbackground=self.fg_color
                    )
                    e.grid(row=0, column=i, padx=2)
                    entries.append(e)
                self.list_entries[field] = entries
            else:
                ent = tk.Entry(
                    self.detail_frame,
                    width=60,
                    bg=self.entry_bg,
                    fg=self.fg_color,
                    insertbackground=self.fg_color
                )
                ent.grid(row=starting_row + 1, column=0, sticky=tk.W+tk.E)
                self.entries[field] = ent

            starting_row += 2

        # Footer (barre de statut en bas)
        self.status_var = tk.StringVar()
        self.status_var.set(texts['status_no_file'])
        lbl_status = tk.Label(
            self,
            textvariable=self.status_var,
            relief=tk.SUNKEN,
            anchor=tk.W,
            bg=self.bg_color,
            fg=self.fg_color
        )
        lbl_status.pack(fill=tk.X, side=tk.BOTTOM)

    # ---------- Parsing fichier ----------
    def parse_file(self, path):
        """Lit le fichier et retourne un OrderedDict."""
        data = OrderedDict()
        current_section = None
        section_data = {}

        with open(path, "r", encoding="utf-8") as f:
            for raw_line in f:
                line = raw_line.strip()
                if not line:
                    continue
                if line.startswith("[") and line.endswith("]"):
                    if current_section is not None:
                        data[current_section] = section_data
                    current_section = line[1:-1]
                    section_data = {}
                else:
                    if "=" in line and current_section is not None:
                        key, val = line.split("=", 1)
                        section_data[key.strip()] = val.strip()
            if current_section is not None:
                data[current_section] = section_data

        return data

    def write_file(self, path, data):
        """Réécrit toutes les sections dans le fichier de sortie."""
        with open(path, "w", encoding="utf-8") as f:
            for section_name, attrs in data.items():
                f.write(f"[{section_name}]\n")
                for key, val in attrs.items():
                    f.write(f"{key}={val}\n")
                f.write("\n")

    # ---------- Gestion de l’interface ----------
    def load_file_dialog(self):
        """Ouvre un fichier, parse, et remplit la liste des sections."""
        file_path = filedialog.askopenfilename(
            title=texts['load_file'],
            filetypes=[("Fichiers texte", "*.txt *.ini *.cfg"), ("Tous les fichiers", "*.*")]
        )
        if not file_path:
            return

        try:
            parsed = self.parse_file(file_path)
        except Exception as e:
            messagebox.showerror(
                texts['warning_no_file'],
                f"Impossible de lire le fichier :\n{e}"
            )
            return

        self.data = parsed
        self.current_file = file_path
        self.current_section = None

        self.listbox.delete(0, tk.END)
        for section in self.data.keys():
            self.listbox.insert(tk.END, section)

        self.status_var.set(f"{file_path} ({len(self.data)} sections)")
        self.clear_detail_fields()

    def clear_detail_fields(self):
        """Vide tous les champs d’édition (Entries)."""
        for widget in self.entries.values():
            widget.delete(0, tk.END)
        for entry_list in self.list_entries.values():
            for e in entry_list:
                e.delete(0, tk.END)

    def on_select_section(self, evt):
        """Affiche les valeurs de la section sélectionnée."""
        if not self.listbox.curselection():
            return
        index = int(self.listbox.curselection()[0])
        section_name = self.listbox.get(index)
        self.current_section = section_name

        attrs = self.data.get(section_name, {})

        # Champs simples
        for field, widget in self.entries.items():
            value = attrs.get(field, "")
            widget.delete(0, tk.END)
            widget.insert(0, value)

        # Champs liste
        for field, entry_list in self.list_entries.items():
            raw_val = attrs.get(field, "")
            parts = raw_val.split(",") if raw_val else []
            for i in range(24):
                e = entry_list[i]
                v = parts[i] if i < len(parts) else '0'
                e.delete(0, tk.END)
                e.insert(0, v)

        self.status_var.set(f"Édition de « {section_name} »")

    def on_save_changes(self):
        """Enregistre les modifications dans self.data puis dans le fichier."""
        if not self.current_file:
            messagebox.showwarning(
                texts['warning_no_file'],
                texts['warning_no_file_msg']
            )
            return
        if not self.current_section:
            messagebox.showwarning(
                texts['warning_no_section'],
                texts['warning_no_section_msg']
            )
            return

        # Champs simples
        for field, widget in self.entries.items():
            self.data[self.current_section][field] = widget.get().strip()

        # Champs liste
        for field, entry_list in self.list_entries.items():
            vals = []
            for e in entry_list:
                v = e.get().strip()
                vals.append(v if v != "" else "0")
            self.data[self.current_section][field] = ",".join(vals)

        # Écriture du fichier
        try:
            self.write_file(self.current_file, self.data)
        except Exception as e:
            messagebox.showerror(
                texts['warning_no_file'],
                f"Impossible d'enregistrer :\n{e}"
            )
            return

        self.status_var.set(texts['saved_msg'])

    def on_add_section(self):
        """Ajoute une section par défaut avec tous les champs initialisés à '0'."""
        existing_sections = list(self.data.keys())
        count = len(existing_sections)
        new_section_name = f"ItemSynthesis_{count}"

        default_data = {}
        for field in self.fields:
            if field in ("MaterialType", "MaterialTypeID", "MaterialCount"):
                default_data[field] = ",".join(["0"] * 24)
            else:
                default_data[field] = "0"

        self.data[new_section_name] = default_data
        # Renommer toutes les clés de façon consécutive
        new_data = OrderedDict()
        idx = 0
        for old_name, attrs in self.data.items():
            new_name = f"ItemSynthesis_{idx}"
            new_data[new_name] = attrs
            idx += 1
        self.data = new_data

        # Mettre à jour la Listbox
        self.listbox.delete(0, tk.END)
        for section in self.data.keys():
            self.listbox.insert(tk.END, section)

        # Sélectionner et afficher
        self.current_section = f"ItemSynthesis_{count}"
        self.listbox.selection_set(count)
        self.listbox.see(count)
        for field, widget in self.entries.items():
            widget.delete(0, tk.END)
            widget.insert(0, "0")
        for field, entry_list in self.list_entries.items():
            for e in entry_list:
                e.delete(0, tk.END)
                e.insert(0, "0")

        self.status_var.set(texts['saved_msg'])

    def on_delete_section(self):
        """Supprime la section sélectionnée après confirmation, renumérote et enregistre."""
        if not self.current_section:
            messagebox.showwarning(
                texts['warning_no_section'],
                texts['warning_no_section_msg']
            )
            return

        msg = texts['confirm_delete_msg'].format(section=self.current_section)
        resp = messagebox.askyesno(texts['confirm_delete'], msg)
        if not resp:
            return

        # Supprimer de la donnée
        del self.data[self.current_section]
        # Renommer consécutivement
        new_data = OrderedDict()
        idx = 0
        for old_name, attrs in self.data.items():
            new_name = f"ItemSynthesis_{idx}"
            new_data[new_name] = attrs
            idx += 1
        self.data = new_data

        # Mettre à jour la Listbox
        self.listbox.delete(0, tk.END)
        for section in self.data.keys():
            self.listbox.insert(tk.END, section)

        # Réinitialiser la sélection
        self.current_section = None
        self.clear_detail_fields()

        # Enregistrer dans le fichier
        try:
            self.write_file(self.current_file, self.data)
        except Exception as e:
            messagebox.showerror(
                texts['warning_no_file'],
                f"Impossible d'enregistrer après suppression :\n{e}"
            )
            return

        self.status_var.set(texts['deleted_msg'])

# ──────────────────────────────────────────────────────────────────────────────
# OBELISKE EDITOR (onglet "EditObelisk")
# ──────────────────────────────────────────────────────────────────────────────

class ObeliskEditorFrame(tk.Frame):
    """
    Frame qui contient l’éditeur visuel Obelisk (zones, obelisk, mobs, bothmobs, portails)
    dans l’onglet EditObelisk.
    """
    def __init__(self, parent):
        super().__init__(parent)
        self.apply_dark_mode()

        # Style sombre pour le Treeview
        style = ttk.Style(self)
        style.theme_use("clam")
        style.configure("Treeview",
                        background="#2e2e2e",
                        foreground="white",
                        fieldbackground="#2e2e2e",
                        rowheight=24,
                        bordercolor="#444",
                        borderwidth=1)
        style.map("Treeview", background=[("selected", "#6a6a6a")])

        # Arborescence
        self.tree = ttk.Treeview(self)
        self.tree.heading("#0", text="Zones / Obélisques / BothMobs / Portails", anchor='w')
        self.tree.pack(fill=tk.BOTH, expand=True, side=tk.LEFT)
        # Tags couleur
        self.tree.tag_configure("zone_tag", background="#3e3e3e", foreground="white")
        self.tree.tag_configure("obelisk_tag", background="#2e2e2e", foreground="white")
        self.tree.tag_configure("mob_tag", background="#2e2e2e", foreground="white")
        self.tree.tag_configure("bothmob_tag", background="#3e3e3e", foreground="white")
        self.tree.tag_configure("portal_tag", background="#3e3e3e", foreground="white")
        self.tree.tag_configure("portal_entry_tag", background="#2e2e2e", foreground="white")

        # Cadre de droite : formulaires et boutons
        self.details_frame = tk.Frame(self, bg="#2e2e2e", padx=10, pady=10)
        self.details_frame.pack(fill=tk.BOTH, expand=True, side=tk.RIGHT)

        # Boutons supérieurs
        btn_frame = tk.Frame(self.details_frame, bg="#2e2e2e")
        btn_frame.pack(anchor='nw')
        self.add_zone_button = tk.Button(
            btn_frame, text="➕ Ajouter Zone", command=self.add_zone,
            bg="#444", fg="white", activebackground="#555", activeforeground="white"
        )
        self.add_zone_button.pack(side=tk.LEFT, padx=(0, 5))

        self.add_bothmob_button = tk.Button(
            btn_frame, text="➕ Ajouter BothMob", command=self.add_bothmob,
            bg="#444", fg="white", activebackground="#555", activeforeground="white"
        )
        self.add_bothmob_button.pack(side=tk.LEFT, padx=(0, 5))

        self.load_button = tk.Button(
            btn_frame, text="ðﾟﾓﾂ Charger Fichier", command=self.load_file,
            bg="#444", fg="white", activebackground="#555", activeforeground="white"
        )
        self.load_button.pack(side=tk.LEFT, padx=(0, 5))

        self.save_button = tk.Button(
            btn_frame, text="ðﾟﾒﾾ Sauvegarder Fichier", command=self.save_file,
            bg="#444", fg="white", activebackground="#555", activeforeground="white"
        )
        self.save_button.pack(side=tk.LEFT)

        ttk.Separator(self.details_frame, orient=tk.HORIZONTAL).pack(fill=tk.X, pady=5)

        # Formulaire dynamique
        self.detail_widgets = {}
        self.current_selection = None
        self.form_frame = tk.Frame(self.details_frame, bg="#2e2e2e")
        self.form_frame.pack(fill=tk.BOTH, expand=True, anchor='nw')

        # Lier sélection Treeview
        self.tree.bind("<<TreeviewSelect>>", self.on_tree_select)

        # Données en mémoire
        self.data = {}      # Zones → Obelisks → Mobs
        self.bothmobs = {}  # BothMobs → params, mobs enfants, itemdrops
        self.portals = {}   # Portails → paramètres + listes light/fury

    def apply_dark_mode(self):
        """Applique fond sombre au frame."""
        self.configure(bg="#2e2e2e")

    def create_label(self, parent, text, row=None, column=None, **kwargs):
        """Crée un Label sombre."""
        lbl = tk.Label(parent, text=text, bg="#2e2e2e", fg="white", **kwargs)
        if row is not None and column is not None:
            lbl.grid(row=row, column=column, sticky='e', padx=2, pady=2)
        return lbl

    def create_entry(self, parent, textvariable, row, column):
        """Crée un Entry sombre."""
        entry = tk.Entry(parent, textvariable=textvariable, bg="#3e3e3e", fg="white", insertbackground="white")
        entry.grid(row=row, column=column, sticky='w', padx=2, pady=2)
        return entry

    def create_button(self, parent, text, command, row=None, column=None, colspan=1, pady=2):
        """Crée un bouton sombre."""
        btn = tk.Button(parent, text=text, command=command,
                        bg="#444", fg="white", activebackground="#555", activeforeground="white")
        if row is not None and column is not None:
            btn.grid(row=row, column=column, columnspan=colspan, pady=pady, sticky='we', padx=2)
        return btn

    def load_file(self):
        """Charge un fichier texte, parse, remplit l'arborescence."""
        filepath = filedialog.askopenfilename(
            title="Sélectionnez le fichier",
            filetypes=[("Text Files", "*.ini"), ("All Files", "*.*")]
        )
        if not filepath:
            return
        try:
            with open(filepath, 'r', encoding='utf-8') as f:
                content = f.read()
            self.parse_content(content)
            self.populate_tree()
            self.current_file = filepath
            messagebox.showinfo("Chargé", f"Fichier chargé :\n{filepath}")
        except Exception as e:
            messagebox.showerror("Erreur", f"Impossible de charger : {e}")

    def parse_content(self, content):
        """
        Parse le contenu brut et remplit :
          - self.data   : Zones, Obelisks, Mobs
          - self.bothmobs : BothMobs, mobs enfants, itemdrops
          - self.portals  : Portails, entrées light/fury
        """
        self.data = {}
        self.bothmobs = {}
        self.portals = {}

        current_zone = None
        current_ob = None
        current_both = None
        current_portal = None

        zone_pattern = re.compile(r'^\s*\[(zone_[0-9]+)\]\s*$', re.IGNORECASE)
        ob_pattern = re.compile(
            r'^\s*\[(Obelisk_[0-9]+)\]\s*=\s*'
            r'\(\s*([0-9]+(?:\.[0-9]+)?)\s*,\s*([0-9]+(?:\.[0-9]+)?)\s*,\s*([0-9]+(?:\.[0-9]+)?)\s*\)\s*,\s*'
            r'\(\s*([0-9]+)\s*,\s*([0-9]+)\s*,\s*([0-9]+)\s*\)\s*,\s*([0-2])\s*$'
        )
        mob_pattern = re.compile(
            r'^\s*\[Mob\]\s*=\s*'
            r'\(\s*([0-9]+(?:\.[0-9]+)?)\s*,\s*([0-9]+(?:\.[0-9]+)?)\s*,\s*([0-9]+(?:\.[0-9]+)?)\s*,\s*([0-9]+(?:\.[0-9]+)?)\s*\)\s*,\s*'
            r'\(\s*([0-9]+)\s*,\s*([0-9]+)\s*,\s*([0-9]+)\s*\)\s*,\s*([0-9]+)\s*$'
        )
        bothmob_pattern = re.compile(
            r'^\s*\[(bothmob_[0-9]+)\s*,\s*'
            r'\(\s*([0-9]+)\s*,\s*([0-9]+)\s*,\s*([0-9]+)\s*,\s*'
            r'\(\s*([0-9]+)\s*,\s*([0-9]+)\)\s*\)\]\s*=\s*'
            r'\(\s*([0-9]+(?:\.[0-9]+)?)\s*,\s*([0-9]+(?:\.[0-9]+)?)\s*,\s*([0-9]+(?:\.[0-9]+)?)\s*\)\s*,\s*'
            r'\(\s*([0-9]+)\s*,\s*([0-9]+)\s*\)\s*$'
        )
        portal_header = re.compile(
            r'^\s*\[(Portal_[0-9]+)\s*,\s*\(\s*([0-9]+)\s*,\s*([0-9]+)\s*\)\s*\]\s*$'
        )
        portal_line = re.compile(r'^\s*\[(light|fury)\]\s*=\s*(.*)$')

        for line in content.splitlines():
            line = line.strip()
            if not line:
                continue

            zm = zone_pattern.match(line)
            if zm:
                zone_key = zm.group(1)
                self.data[zone_key] = {}
                current_zone = zone_key
                current_ob = None
                current_both = None
                current_portal = None
                continue

            om = ob_pattern.match(line)
            if om and current_zone:
                ob_key = om.group(1)
                x, y, z = float(om.group(2)), float(om.group(3)), float(om.group(4))
                mN, mL, mF = int(om.group(5)), int(om.group(6)), int(om.group(7))
                default_faction = int(om.group(8))
                self.data[current_zone][ob_key] = {
                    "position": [x, y, z],
                    "mob_ids": [mN, mL, mF],
                    "default_faction": default_faction,
                    "mobs": []
                }
                current_ob = ob_key
                current_both = None
                current_portal = None
                continue

            bm = bothmob_pattern.match(line)
            if bm:
                both_key = bm.group(1)
                map_no = int(bm.group(2))
                change_count = int(bm.group(3))
                respawn = int(bm.group(4))
                portal_map = int(bm.group(5))
                portal_id = int(bm.group(6))
                bx, by, bz = float(bm.group(7)), float(bm.group(8)), float(bm.group(9))
                idA, idB = int(bm.group(10)), int(bm.group(11))
                self.bothmobs[both_key] = {
                    "params": [map_no, change_count, respawn, [portal_map, portal_id]],
                    "position": [bx, by, bz],
                    "ids": [idA, idB],
                    "mobs": [],
                    "itemdrops": []
                }
                current_zone = None
                current_ob = None
                current_both = both_key
                current_portal = None
                continue

            ph = portal_header.match(line)
            if ph:
                portal_key = ph.group(1)
                map_no = int(ph.group(2))
                portal_id = int(ph.group(3))
                self.portals[portal_key] = {
                    "mapno": map_no,
                    "portalid": portal_id,
                    "light": [],
                    "fury": []
                }
                current_zone = None
                current_ob = None
                current_both = None
                current_portal = portal_key
                continue

            pl = portal_line.match(line)
            if pl and current_portal:
                typ = pl.group(1)
                rest = pl.group(2)
                pairs = re.findall(r'\(\s*([0-9]+)\s*,\s*(Obelisk_[0-9]+)\s*\)', rest)
                lst = [(int(m), ob) for (m, ob) in pairs]
                self.portals[current_portal][typ] = lst
                continue

            mm = mob_pattern.match(line)
            if mm:
                mx, my, mz, mr = (float(mm.group(i)) for i in range(1, 5))
                idN, idL, idF = int(mm.group(5)), int(mm.group(6)), int(mm.group(7))
                count = int(mm.group(8))
                mob_entry = {
                    "position": [mx, my, mz],
                    "radius": mr,
                    "mob_ids": [idN, idL, idF],
                    "count": count
                }
                if current_ob:
                    self.data[current_zone][current_ob]["mobs"].append(mob_entry)
                elif current_both:
                    if idN == 0 and idL == 0 and idF == 1972:
                        self.bothmobs[current_both]["itemdrops"].append(mob_entry)
                    else:
                        self.bothmobs[current_both]["mobs"].append(mob_entry)
                continue

    def populate_tree(self):
        """Met à jour l'arborescence selon les données en mémoire."""
        self.tree.delete(*self.tree.get_children())

        # 1) Zones → Obelisks → Mobs
        for zone_key, obelisks in sorted(self.data.items()):
            z_id = self.tree.insert(
                "", tk.END, text=zone_key, open=True,
                values=("zone", zone_key), tags=("zone_tag",)
            )
            for ob_key, ob_data in sorted(obelisks.items()):
                o_id = self.tree.insert(
                    z_id, tk.END, text=ob_key, open=False,
                    values=("obelisk", zone_key, ob_key), tags=("obelisk_tag",)
                )
                for idx, mob in enumerate(ob_data["mobs"]):
                    mob_text = f"Mob #{idx+1} (IDs={tuple(mob['mob_ids'])})"
                    self.tree.insert(
                        o_id, tk.END, text=mob_text,
                        values=("mob", zone_key, ob_key, idx), tags=("mob_tag",)
                    )

        # 2) BothMobs
        if self.bothmobs:
            both_root = self.tree.insert("", tk.END, text="BothMobs", open=True,
                                        values=("bothroot",), tags=("bothmob_tag",))
            for both_key, both_data in sorted(self.bothmobs.items()):
                header = f" Params={tuple(both_data['params'])} Pos={tuple(both_data['position'])} IDs={tuple(both_data['ids'])}"
                b_id = self.tree.insert(
                    both_root, tk.END, text=f"{both_key}{header}", open=False,
                    values=("bothmob", both_key), tags=("bothmob_tag",)
                )
                for idx, mob in enumerate(both_data["mobs"]):
                    mob_text = f"Mob #{idx+1} (IDs={tuple(mob['mob_ids'])})"
                    self.tree.insert(
                        b_id, tk.END, text=mob_text,
                        values=("bothmob_mob", both_key, idx), tags=("mob_tag",)
                    )
                for idx, item in enumerate(both_data["itemdrops"]):
                    item_text = f"ItemDrop #{idx+1} (IDs={tuple(item['mob_ids'])})"
                    self.tree.insert(
                        b_id, tk.END, text=item_text,
                        values=("bothmob_item", both_key, idx), tags=("mob_tag",)
                    )

        # 3) Portails
        if self.portals:
            portal_root = self.tree.insert("", tk.END, text="Portails", open=True,
                                           values=("portalroot",), tags=("portal_tag",))
            for p_key, p_data in sorted(self.portals.items()):
                header = f" (Map={p_data['mapno']}, ID={p_data['portalid']})"
                p_id = self.tree.insert(
                    portal_root, tk.END, text=f"{p_key}{header}", open=False,
                    values=("portal", p_key), tags=("portal_tag",)
                )
                if p_data["light"]:
                    light_id = self.tree.insert(
                        p_id, tk.END, text="► light", open=True,
                        values=("portal_light", p_key), tags=("portal_entry_tag",)
                    )
                    for idx, (m, ob) in enumerate(p_data["light"]):
                        entry_text = f"{idx+1}. (Map {m}, {ob})"
                        self.tree.insert(
                            light_id, tk.END, text=entry_text,
                            values=("portal_light_entry", p_key, idx), tags=("portal_entry_tag",)
                        )
                if p_data["fury"]:
                    fury_id = self.tree.insert(
                        p_id, tk.END, text="► fury", open=True,
                        values=("portal_fury", p_key), tags=("portal_entry_tag",)
                    )
                    for idx, (m, ob) in enumerate(p_data["fury"]):
                        entry_text = f"{idx+1}. (Map {m}, {ob})"
                        self.tree.insert(
                            fury_id, tk.END, text=entry_text,
                            values=("portal_fury_entry", p_key, idx), tags=("portal_entry_tag",)
                        )

    def on_tree_select(self, event):
        """
        Lorsqu'un élément est sélectionné, on vide l'ancien formulaire
        et on affiche le bon formulaire selon le type.
        """
        selected = self.tree.focus()
        values = self.tree.item(selected, "values")

        # Vider l'ancien formulaire
        for w in self.form_frame.winfo_children():
            w.destroy()
        self.detail_widgets.clear()
        self.current_selection = None

        if not values:
            return

        typ = values[0]
        if typ == "zone":
            _, zone_key = values
            self.show_zone_form(zone_key)
            self.current_selection = ("zone", zone_key)

        elif typ == "obelisk":
            _, zone_key, ob_key = values
            self.show_obelisk_form(zone_key, ob_key)
            self.current_selection = ("obelisk", zone_key, ob_key)

        elif typ == "mob":
            _, zone_key, ob_key, mob_idx = values
            mob_idx = int(mob_idx)
            self.show_mob_form(zone_key, ob_key, mob_idx)
            self.current_selection = ("mob", zone_key, ob_key, mob_idx)

        elif typ == "bothroot":
            self.show_bothmob_root_form()

        elif typ == "bothmob":
            _, both_key = values
            self.show_bothmob_form(both_key)
            self.current_selection = ("bothmob", both_key)

        elif typ == "bothmob_mob":
            _, both_key, mob_idx = values
            mob_idx = int(mob_idx)
            self.show_bothmob_mob_form(both_key, mob_idx, is_item=False)
            self.current_selection = ("bothmob_mob", both_key, mob_idx)

        elif typ == "bothmob_item":
            _, both_key, mob_idx = values
            mob_idx = int(mob_idx)
            self.show_bothmob_mob_form(both_key, mob_idx, is_item=True)
            self.current_selection = ("bothmob_item", both_key, mob_idx)

        elif typ == "portal":
            _, p_key = values
            self.show_portal_form(p_key)
            self.current_selection = ("portal", p_key)

        elif typ == "portal_light_entry":
            _, p_key, idx = values
            idx = int(idx)
            self.show_portal_entry_form(p_key, idx, is_light=True)
            self.current_selection = ("portal_light_entry", p_key, idx)

        elif typ == "portal_fury_entry":
            _, p_key, idx = values
            idx = int(idx)
            self.show_portal_entry_form(p_key, idx, is_light=False)
            self.current_selection = ("portal_fury_entry", p_key, idx)

    # ---------- Zone ----------
    def show_zone_form(self, zone_key):
        """Formulaire pour une zone."""
        tk.Label(
            self.form_frame, text=f"Zone : {zone_key}", font=("Arial", 14, "bold"),
            bg="#2e2e2e", fg="white"
        ).grid(row=0, column=0, columnspan=2, sticky='w', pady=(0, 10))

        self.create_button(
            self.form_frame, "➕ Ajouter Obélisque",
            command=lambda: self.add_obelisk(zone_key),
            row=1, column=0, colspan=2
        )
        self.create_button(
            self.form_frame, "❌ Supprimer Zone",
            command=lambda: self.delete_zone(zone_key),
            row=2, column=0, colspan=2, pady=10
        )

    def add_zone(self):
        """Ajoute une zone demandée par l'utilisateur."""
        new_name = simpledialog.askstring(
            "Nouvelle Zone", "Entrez le nom de la nouvelle zone (ex: zone_99) :",
            parent=self
        )
        if not new_name:
            return
        new_name = new_name.strip()
        if new_name in self.data:
            messagebox.showerror("Erreur", "Cette zone existe déjà.")
            return
        self.data[new_name] = {}
        self.populate_tree()

    def delete_zone(self, zone_key):
        """Supprime une zone après confirmation."""
        resp = messagebox.askyesno(
            "Confirmation",
            f"Voulez-vous vraiment supprimer la zone « {zone_key} » ?"
        )
        if not resp:
            return
        del self.data[zone_key]
        messagebox.showinfo("Supprimé", f"Zone « {zone_key} » supprimée.")
        self.populate_tree()

    # ---------- Obélisque ----------
    def show_obelisk_form(self, zone_key, ob_key):
        """Formulaire pour éditer ou supprimer un obélisque et gérer ses mobs."""
        data = self.data[zone_key][ob_key]
        tk.Label(
            self.form_frame, text=f"Obélisque : {ob_key}", font=("Arial", 14, "bold"),
            bg="#2e2e2e", fg="white"
        ).grid(row=0, column=0, columnspan=2, sticky='w')

        # Position
        self.create_label(self.form_frame, "Position X :", row=1, column=0)
        pos_x_var = tk.StringVar(value=str(data["position"][0]))
        self.create_entry(self.form_frame, pos_x_var, row=1, column=1)
        self.create_label(self.form_frame, "Position Y :", row=2, column=0)
        pos_y_var = tk.StringVar(value=str(data["position"][1]))
        self.create_entry(self.form_frame, pos_y_var, row=2, column=1)
        self.create_label(self.form_frame, "Position Z :", row=3, column=0)
        pos_z_var = tk.StringVar(value=str(data["position"][2]))
        self.create_entry(self.form_frame, pos_z_var, row=3, column=1)

        # MobIDs
        self.create_label(self.form_frame, "MobID Neutral :", row=4, column=0)
        mN_var = tk.StringVar(value=str(data["mob_ids"][0]))
        self.create_entry(self.form_frame, mN_var, row=4, column=1)
        self.create_label(self.form_frame, "MobID Light :", row=5, column=0)
        mL_var = tk.StringVar(value=str(data["mob_ids"][1]))
        self.create_entry(self.form_frame, mL_var, row=5, column=1)
        self.create_label(self.form_frame, "MobID Fury :", row=6, column=0)
        mF_var = tk.StringVar(value=str(data["mob_ids"][2]))
        self.create_entry(self.form_frame, mF_var, row=6, column=1)

        # Faction
        self.create_label(self.form_frame, "Faction :", row=7, column=0)
        faction_var = tk.StringVar(value=str(data["default_faction"]))
        faction_cb = ttk.Combobox(
            self.form_frame, textvariable=faction_var,
            values=["0", "1", "2"], width=3, state="readonly"
        )
        faction_cb.grid(row=7, column=1, sticky='w', padx=2, pady=2)

        # Sauvegarder / Supprimer obélisque
        self.create_button(
            self.form_frame, "ðﾟﾒﾾ Enregistrer Obélisque",
            command=lambda: self.save_obelisk(
                zone_key, ob_key,
                pos_x_var.get(), pos_y_var.get(), pos_z_var.get(),
                mN_var.get(), mL_var.get(), mF_var.get(),
                faction_var.get()
            ),
            row=8, column=0, colspan=2, pady=10
        )
        self.create_button(
            self.form_frame, "❌ Supprimer Obélisque",
            command=lambda: self.delete_obelisk(zone_key, ob_key),
            row=9, column=0, colspan=2, pady=10
        )

        # Séparateur
        ttk.Separator(self.form_frame, orient=tk.HORIZONTAL).grid(
            row=10, column=0, columnspan=2, sticky="we", pady=10
        )

        # Liste des mobs
        tk.Label(
            self.form_frame, text="Mobs attachés :", font=("Arial", 12, "bold"),
            bg="#2e2e2e", fg="white"
        ).grid(row=11, column=0, columnspan=2, sticky='w')

        tree_frame = tk.Frame(self.form_frame, bg="#2e2e2e")
        tree_frame.grid(row=12, column=0, columnspan=2, sticky="nsew")
        self.form_frame.rowconfigure(12, weight=1)

        columns = ("position", "radius", "mob_ids", "count")
        mob_table = ttk.Treeview(tree_frame, columns=columns, show="headings", height=8)
        mob_table.heading("position", text="Position (X,Y,Z)")
        mob_table.heading("radius", text="Radius")
        mob_table.heading("mob_ids", text="Mob IDs")
        mob_table.heading("count", text="Count")
        mob_table.pack(fill=tk.BOTH, expand=True, side=tk.LEFT)

        scrollbar = ttk.Scrollbar(tree_frame, orient="vertical", command=mob_table.yview)
        mob_table.configure(yscroll=scrollbar.set)
        scrollbar.pack(fill="y", side=tk.RIGHT)

        for idx, mob in enumerate(data["mobs"]):
            pos = ",".join(map(str, mob["position"]))
            mob_ids = ",".join(map(str, mob["mob_ids"]))
            mob_table.insert(
                "", tk.END, iid=str(idx),
                values=(pos, str(mob["radius"]), mob_ids, str(mob["count"]))
            )

        mob_table.bind(
            "<Double-1>",
            lambda e: self.on_mob_double_click(e, zone_key, ob_key, mob_table)
        )

        self.create_button(
            self.form_frame, "➕ Ajouter Mob",
            command=lambda: self.add_mob(zone_key, ob_key),
            row=13, column=0, colspan=2, pady=10
        )

    def add_obelisk(self, zone_key):
        """Ajoute un nouvel obélisque dans la zone."""
        existing = list(self.data[zone_key].keys())
        num = 1
        while True:
            new_name = f"Obelisk_{num:03d}"
            if new_name not in existing:
                break
            num += 1
        self.data[zone_key][new_name] = {
            "position": [0.0, 0.0, 0.0],
            "mob_ids": [0, 0, 0],
            "default_faction": 0,
            "mobs": []
        }
        self.populate_tree()

    def save_obelisk(self, zone_key, ob_key, x, y, z, mN, mL, mF, faction):
        """Sauvegarde les modifications d'un obélisque."""
        try:
            x_f, y_f, z_f = float(x), float(y), float(z)
            iN, iL, iF = int(mN), int(mL), int(mF)
            fct = int(faction)
        except ValueError:
            messagebox.showerror("Erreur", "Valeurs invalides pour l'obélisque.")
            return
        self.data[zone_key][ob_key]["position"] = [x_f, y_f, z_f]
        self.data[zone_key][ob_key]["mob_ids"] = [iN, iL, iF]
        self.data[zone_key][ob_key]["default_faction"] = fct
        messagebox.showinfo("Enregistré", f"Obélisque « {ob_key} » mis à jour.")
        self.populate_tree()

    def delete_obelisk(self, zone_key, ob_key):
        """Supprime un obélisque après confirmation."""
        resp = messagebox.askyesno("Confirmation", f"Supprimer l'obélisque « {ob_key} » ?")
        if not resp:
            return
        del self.data[zone_key][ob_key]
        messagebox.showinfo("Supprimé", f"Obélisque « {ob_key} » supprimé.")
        self.populate_tree()

    def add_mob(self, zone_key, ob_key):
        """Ajoute un mob vide sous un obélisque."""
        new_mob = {
            "position": [0.0, 0.0, 0.0],
            "radius": 0.0,
            "mob_ids": [0, 0, 0],
            "count": 1
        }
        self.data[zone_key][ob_key]["mobs"].append(new_mob)
        self.populate_tree()

    def show_mob_form(self, zone_key, ob_key, mob_idx):
        """Formulaire pour éditer ou supprimer un mob d'un obélisque."""
        mob = self.data[zone_key][ob_key]["mobs"][mob_idx]
        tk.Label(
            self.form_frame, text=f"Mob #{mob_idx+1} (Obélisque {ob_key})",
            font=("Arial", 14, "bold"), bg="#2e2e2e", fg="white"
        ).grid(row=0, column=0, columnspan=2, sticky='w')

        self.create_label(self.form_frame, "Position X :", row=1, column=0)
        mx_var = tk.StringVar(value=str(mob["position"][0]))
        self.create_entry(self.form_frame, mx_var, row=1, column=1)

        self.create_label(self.form_frame, "Position Y :", row=2, column=0)
        my_var = tk.StringVar(value=str(mob["position"][1]))
        self.create_entry(self.form_frame, my_var, row=2, column=1)

        self.create_label(self.form_frame, "Position Z :", row=3, column=0)
        mz_var = tk.StringVar(value=str(mob["position"][2]))
        self.create_entry(self.form_frame, mz_var, row=3, column=1)

        self.create_label(self.form_frame, "Radius :", row=4, column=0)
        mr_var = tk.StringVar(value=str(mob["radius"]))
        self.create_entry(self.form_frame, mr_var, row=4, column=1)

        self.create_label(self.form_frame, "MobID Neutral :", row=5, column=0)
        idN_var = tk.StringVar(value=str(mob["mob_ids"][0]))
        self.create_entry(self.form_frame, idN_var, row=5, column=1)

        self.create_label(self.form_frame, "MobID Light :", row=6, column=0)
        idL_var = tk.StringVar(value=str(mob["mob_ids"][1]))
        self.create_entry(self.form_frame, idL_var, row=6, column=1)

        self.create_label(self.form_frame, "MobID Fury :", row=7, column=0)
        idF_var = tk.StringVar(value=str(mob["mob_ids"][2]))
        self.create_entry(self.form_frame, idF_var, row=7, column=1)

        self.create_label(self.form_frame, "Count :", row=8, column=0)
        count_var = tk.StringVar(value=str(mob["count"]))
        self.create_entry(self.form_frame, count_var, row=8, column=1)

        self.create_button(
            self.form_frame, "ðﾟﾒﾾ Enregistrer Mob",
            command=lambda: self.save_mob(
                zone_key, ob_key, mob_idx,
                mx_var.get(), my_var.get(), mz_var.get(),
                mr_var.get(), idN_var.get(), idL_var.get(), idF_var.get(),
                count_var.get()
            ),
            row=9, column=0, colspan=2, pady=10
        )
        self.create_button(
            self.form_frame, "❌ Supprimer Mob",
            command=lambda: self.delete_mob(zone_key, ob_key, mob_idx),
            row=10, column=0, colspan=2, pady=2
        )

    def save_mob(self, zone_key, ob_key, mob_idx, x, y, z, r, iN, iL, iF, count):
        """Sauvegarde les modifications d'un mob."""
        try:
            x_f, y_f, z_f = float(x), float(y), float(z)
            r_f = float(r)
            idN_i, idL_i, idF_i = int(iN), int(iL), int(iF)
            cnt = int(count)
        except ValueError:
            messagebox.showerror("Erreur", "Valeurs invalides pour le mob.")
            return
        mob = self.data[zone_key][ob_key]["mobs"][mob_idx]
        mob["position"] = [x_f, y_f, z_f]
        mob["radius"] = r_f
        mob["mob_ids"] = [idN_i, idL_i, idF_i]
        mob["count"] = cnt
        messagebox.showinfo("Enregistré", f"Mob #{mob_idx+1} mis à jour.")
        self.populate_tree()

    def delete_mob(self, zone_key, ob_key, mob_idx):
        """Supprime un mob après confirmation."""
        resp = messagebox.askyesno("Confirmation", "Supprimer ce mob ?")
        if not resp:
            return
        del self.data[zone_key][ob_key]["mobs"][mob_idx]
        messagebox.showinfo("Supprimé", "Mob supprimé.")
        self.populate_tree()

    def on_mob_double_click(self, event, zone_key, ob_key, mob_table):
        """
        Double-clic dans la table des mobs : sélectionne l'élément correspondant
        dans l'arborescence pour ouvrir le formulaire d'édition.
        """
        row_id = mob_table.focus()
        if not row_id:
            return
        mob_idx = int(row_id)
        # On ne se base que sur les nœuds dont values[0] == "zone"
        for z in self.tree.get_children():
            vals_z = self.tree.item(z, "values")
            if len(vals_z) >= 2 and vals_z[0] == "zone" and vals_z[1] == zone_key:
                for o in self.tree.get_children(z):
                    vals_o = self.tree.item(o, "values")
                    if len(vals_o) >= 3 and vals_o[0] == "obelisk" and vals_o[1] == zone_key and vals_o[2] == ob_key:
                        for m in self.tree.get_children(o):
                            vals_m = self.tree.item(m, "values")
                            # vals_m doit avoir au moins 4 éléments et vals_m[3] correspondre à mob_idx
                            if len(vals_m) >= 4 and vals_m[0] == "mob" and int(vals_m[3]) == mob_idx:
                                self.tree.selection_set(m)
                                self.tree.see(m)
                                return

    # ---------- BothMobs ----------
    def show_bothmob_root_form(self):
        """Formulaire minimal pour racine BothMobs."""
        tk.Label(
            self.form_frame, text="BothMobs", font=("Arial", 14, "bold"),
            bg="#2e2e2e", fg="white"
        ).grid(row=0, column=0, columnspan=2, sticky='w', pady=(0, 10))

        self.create_button(
            self.form_frame, "➕ Ajouter BothMob",
            command=self.add_bothmob,
            row=1, column=0, colspan=2, pady=5
        )

    def show_bothmob_form(self, both_key):
        """
        Formulaire pour éditer un BothMob :
          - Paramètres (mapno, change_count, respawn, portal_map, portal_id)
          - Position du boss
          - IDs du boss
          - Bouton Enregistrer / Supprimer BothMob
          - Boutons Ajouter Mob / Ajouter ItemDrop
        """
        data = self.bothmobs[both_key]
        tk.Label(
            self.form_frame, text=f"BothMob : {both_key}", font=("Arial", 14, "bold"),
            bg="#2e2e2e", fg="white"
        ).grid(row=0, column=0, columnspan=2, sticky='w')

        # MapNo
        self.create_label(self.form_frame, "MapNo :", row=1, column=0)
        mapno_var = tk.StringVar(value=str(data["params"][0]))
        self.create_entry(self.form_frame, mapno_var, row=1, column=1)

        # ChangeCount
        self.create_label(self.form_frame, "ChangeCount :", row=2, column=0)
        cc_var = tk.StringVar(value=str(data["params"][1]))
        self.create_entry(self.form_frame, cc_var, row=2, column=1)

        # Respawn
        self.create_label(self.form_frame, "Respawn (s) :", row=3, column=0)
        resp_var = tk.StringVar(value=str(data["params"][2]))
        self.create_entry(self.form_frame, resp_var, row=3, column=1)

        # Portal Map
        self.create_label(self.form_frame, "Portal Map :", row=4, column=0)
        pmap_var = tk.StringVar(value=str(data["params"][3][0]))
        self.create_entry(self.form_frame, pmap_var, row=4, column=1)

        # Portal ID
        self.create_label(self.form_frame, "Portal ID :", row=5, column=0)
        pid_var = tk.StringVar(value=str(data["params"][3][1]))
        self.create_entry(self.form_frame, pid_var, row=5, column=1)

        # Position du boss
        self.create_label(self.form_frame, "Position X :", row=6, column=0)
        bx_var = tk.StringVar(value=str(data["position"][0]))
        self.create_entry(self.form_frame, bx_var, row=6, column=1)

        self.create_label(self.form_frame, "Position Y :", row=7, column=0)
        by_var = tk.StringVar(value=str(data["position"][1]))
        self.create_entry(self.form_frame, by_var, row=7, column=1)

        self.create_label(self.form_frame, "Position Z :", row=8, column=0)
        bz_var = tk.StringVar(value=str(data["position"][2]))
        self.create_entry(self.form_frame, bz_var, row=8, column=1)

        # IDs du boss
        self.create_label(self.form_frame, "Boss ID A :", row=9, column=0)
        idA_var = tk.StringVar(value=str(data["ids"][0]))
        self.create_entry(self.form_frame, idA_var, row=9, column=1)

        self.create_label(self.form_frame, "Boss ID B :", row=10, column=0)
        idB_var = tk.StringVar(value=str(data["ids"][1]))
        self.create_entry(self.form_frame, idB_var, row=10, column=1)

        # Boutons Enregistrer / Supprimer BothMob
        self.create_button(
            self.form_frame, "ðﾟﾒﾾ Enregistrer BothMob",
            command=lambda: self.save_bothmob_header(
                both_key,
                mapno_var.get(), cc_var.get(), resp_var.get(),
                pmap_var.get(), pid_var.get(),
                bx_var.get(), by_var.get(), bz_var.get(),
                idA_var.get(), idB_var.get()
            ),
            row=11, column=0, colspan=2, pady=10
        )
        self.create_button(
            self.form_frame, "❌ Supprimer BothMob",
            command=lambda: self.delete_bothmob(both_key),
            row=12, column=0, colspan=2, pady=2
        )

        # Séparateur
        ttk.Separator(self.form_frame, orient=tk.HORIZONTAL).grid(
            row=13, column=0, columnspan=2, sticky="we", pady=10
        )

        # Boutons Ajouter Mob / ItemDrop
        self.create_button(
            self.form_frame, "➕ Ajouter Mob",
            command=lambda: self.add_bothmob_mob(both_key, is_item=False),
            row=14, column=0, colspan=2
        )
        self.create_button(
            self.form_frame, "➕ Ajouter ItemDrop",
            command=lambda: self.add_bothmob_mob(both_key, is_item=True),
            row=15, column=0, colspan=2, pady=(5,10)
        )

    def save_bothmob_header(self, both_key,
                             mapno, cc, resp, pmap, pid,
                             bx, by, bz, idA, idB):
        """
        Sauvegarde les paramètres principaux d'un BothMob :
        mapno, change_count, respawn, portal_map, portal_id,
        position du boss, IDs du boss.
        """
        try:
            m = int(mapno)
            c = int(cc)
            r = int(resp)
            pm = int(pmap)
            pi = int(pid)
            bx_f = float(bx)
            by_f = float(by)
            bz_f = float(bz)
            ida = int(idA)
            idb = int(idB)
        except ValueError:
            messagebox.showerror("Erreur", "Valeurs invalides pour BothMob.")
            return

        self.bothmobs[both_key]["params"] = [m, c, r, [pm, pi]]
        self.bothmobs[both_key]["position"] = [bx_f, by_f, bz_f]
        self.bothmobs[both_key]["ids"] = [ida, idb]
        messagebox.showinfo("Enregistré", f"BothMob « {both_key} » mis à jour.")
        self.populate_tree()

    def delete_bothmob(self, both_key):
        """Supprime un BothMob entier après confirmation."""
        resp = messagebox.askyesno("Confirmation", f"Supprimer « {both_key} » ?")
        if not resp:
            return
        del self.bothmobs[both_key]
        messagebox.showinfo("Supprimé", f"{both_key} supprimé.")
        self.populate_tree()

    def add_bothmob(self):
        """
        Demande un nom pour un nouveau BothMob, puis l'ajoute avec valeurs par défaut :
        params=(0,0,0,(0,0)), position=(0,0,0), ids=(0,0), listes vides.
        """
        new_key = simpledialog.askstring(
            "Nouveau BothMob", "Entrez la clé du BothMob (ex: bothmob_010) :",
            parent=self
        )
        if not new_key:
            return
        new_key = new_key.strip()
        if new_key in self.bothmobs:
            messagebox.showerror("Erreur", "Ce BothMob existe déjà.")
            return
        self.bothmobs[new_key] = {
            "params": [0, 0, 0, [0, 0]],
            "position": [0.0, 0.0, 0.0],
            "ids": [0, 0],
            "mobs": [],
            "itemdrops": []
        }
        self.populate_tree()
        messagebox.showinfo("Ajouté", f"BothMob « {new_key} » ajouté.")

    def show_bothmob_mob_form(self, both_key, mob_idx, is_item):
        """
        Formulaire pour éditer un Mob enfant ou un ItemDrop sous un BothMob.
        """
        data = self.bothmobs[both_key]
        mob_list = data["itemdrops"] if is_item else data["mobs"]
        mob = mob_list[mob_idx]
        title = f"{'ItemDrop' if is_item else 'Mob'} #{mob_idx+1} ({both_key})"
        tk.Label(
            self.form_frame, text=title, font=("Arial", 14, "bold"),
            bg="#2e2e2e", fg="white"
        ).grid(row=0, column=0, columnspan=2, sticky='w')

        self.create_label(self.form_frame, "Position X :", row=1, column=0)
        mx_var = tk.StringVar(value=str(mob["position"][0]))
        self.create_entry(self.form_frame, mx_var, row=1, column=1)

        self.create_label(self.form_frame, "Position Y :", row=2, column=0)
        my_var = tk.StringVar(value=str(mob["position"][1]))
        self.create_entry(self.form_frame, my_var, row=2, column=1)

        self.create_label(self.form_frame, "Position Z :", row=3, column=0)
        mz_var = tk.StringVar(value=str(mob["position"][2]))
        self.create_entry(self.form_frame, mz_var, row=3, column=1)

        self.create_label(self.form_frame, "Radius :", row=4, column=0)
        mr_var = tk.StringVar(value=str(mob["radius"]))
        self.create_entry(self.form_frame, mr_var, row=4, column=1)

        self.create_label(self.form_frame, "MobID Neutral :", row=5, column=0)
        idN_var = tk.StringVar(value=str(mob["mob_ids"][0]))
        self.create_entry(self.form_frame, idN_var, row=5, column=1)

        self.create_label(self.form_frame, "MobID Light :", row=6, column=0)
        idL_var = tk.StringVar(value=str(mob["mob_ids"][1]))
        self.create_entry(self.form_frame, idL_var, row=6, column=1)

        self.create_label(self.form_frame, "MobID Fury :", row=7, column=0)
        idF_var = tk.StringVar(value=str(mob["mob_ids"][2]))
        self.create_entry(self.form_frame, idF_var, row=7, column=1)

        self.create_label(self.form_frame, "Count :", row=8, column=0)
        count_var = tk.StringVar(value=str(mob["count"]))
        self.create_entry(self.form_frame, count_var, row=8, column=1)

        self.create_button(
            self.form_frame, "ðﾟﾒﾾ Enregistrer",
            command=lambda: self.save_bothmob_mob(
                both_key, mob_idx, is_item,
                mx_var.get(), my_var.get(), mz_var.get(),
                mr_var.get(), idN_var.get(), idL_var.get(), idF_var.get(),
                count_var.get()
            ),
            row=9, column=0, colspan=2, pady=10
        )
        self.create_button(
            self.form_frame, "❌ Supprimer",
            command=lambda: self.delete_bothmob_mob(both_key, mob_idx, is_item),
            row=10, column=0, colspan=2, pady=2
        )

    def add_bothmob_mob(self, both_key, is_item):
        """
        Ajoute un nouvel enregistrement (Mob ou ItemDrop) avec valeurs par défaut.
        """
        new_mob = {
            "position": [0.0, 0.0, 0.0],
            "radius": 0.0,
            "mob_ids": [0, 0, 0],
            "count": 1
        }
        if is_item:
            self.bothmobs[both_key]["itemdrops"].append(new_mob)
        else:
            self.bothmobs[both_key]["mobs"].append(new_mob)
        self.populate_tree()

    def save_bothmob_mob(self, both_key, mob_idx, is_item,
                         x, y, z, r, iN, iL, iF, count):
        """
        Sauvegarde un mob enfant ou un itemdrop d'un BothMob.
        """
        try:
            x_f, y_f, z_f = float(x), float(y), float(z)
            r_f = float(r)
            idN_i, idL_i, idF_i = int(iN), int(iL), int(iF)
            cnt = int(count)
        except ValueError:
            messagebox.showerror("Erreur", "Valeurs invalides.")
            return
        mob_list = self.bothmobs[both_key]["itemdrops"] if is_item else self.bothmobs[both_key]["mobs"]
        mob = mob_list[mob_idx]
        mob["position"] = [x_f, y_f, z_f]
        mob["radius"] = r_f
        mob["mob_ids"] = [idN_i, idL_i, idF_i]
        mob["count"] = cnt
        messagebox.showinfo("Enregistré", "Entry mise à jour.")
        self.populate_tree()

    def delete_bothmob_mob(self, both_key, mob_idx, is_item):
        """Supprime un mob ou itemdrop sous un BothMob."""
        resp = messagebox.askyesno("Confirmation", "Supprimer cette entrée ?")
        if not resp:
            return
        if is_item:
            del self.bothmobs[both_key]["itemdrops"][mob_idx]
        else:
            del self.bothmobs[both_key]["mobs"][mob_idx]
        messagebox.showinfo("Supprimé", "Entry supprimée.")
        self.populate_tree()

    # ---------- Portails ----------
    def show_portal_form(self, p_key):
        """
        Formulaire pour éditer un Portail :
          - MapNo, PortalID
          - Boutons Enregistrer / Supprimer
          - Sections light et fury : entrées (map cible, obelisk cible)
        """
        data = self.portals[p_key]
        tk.Label(
            self.form_frame, text=f"Portail : {p_key}", font=("Arial", 14, "bold"),
            bg="#2e2e2e", fg="white"
        ).grid(row=0, column=0, columnspan=2, sticky='w')

        # MapNo
        self.create_label(self.form_frame, "MapNo :", row=1, column=0)
        mapno_var = tk.StringVar(value=str(data["mapno"]))
        self.create_entry(self.form_frame, mapno_var, row=1, column=1)

        # PortalID
        self.create_label(self.form_frame, "PortalID :", row=2, column=0)
        pid_var = tk.StringVar(value=str(data["portalid"]))
        self.create_entry(self.form_frame, pid_var, row=2, column=1)

        self.create_button(
            self.form_frame, "ðﾟﾒﾾ Enregistrer Portail",
            command=lambda: self.save_portal_header(p_key, mapno_var.get(), pid_var.get()),
            row=3, column=0, colspan=2, pady=10
        )
        self.create_button(
            self.form_frame, "❌ Supprimer Portail",
            command=lambda: self.delete_portal(p_key),
            row=4, column=0, colspan=2, pady=10
        )

        ttk.Separator(self.form_frame, orient=tk.HORIZONTAL).grid(
            row=5, column=0, columnspan=2, sticky="we", pady=10
        )

        # Section light
        tk.Label(
            self.form_frame, text="light (entrées)", font=("Arial", 12, "bold"),
            bg="#2e2e2e", fg="white"
        ).grid(row=6, column=0, columnspan=2, sticky='w')

        # Afficher chaque entrée light
        for idx, (m, ob) in enumerate(data["light"]):
            row_num = 7 + idx
            tk.Label(self.form_frame, text=f"{idx+1}. Map :", bg="#2e2e2e", fg="white").grid(
                row=row_num, column=0, sticky='e'
            )
            mm_var = tk.StringVar(value=str(m))
            self.create_entry(self.form_frame, mm_var, row=row_num, column=1)
            tk.Label(self.form_frame, text="Obél :", bg="#2e2e2e", fg="white").grid(
                row=row_num, column=2, sticky='e'
            )
            ob_var = tk.StringVar(value=ob)
            ob_entry = tk.Entry(self.form_frame, textvariable=ob_var, bg="#3e3e3e", fg="white", insertbackground="white")
            ob_entry.grid(row=row_num, column=3, sticky='w', padx=2, pady=2)

            # Conserver les StringVar pour sauvegarde
            self.detail_widgets[f"light_map_{idx}"] = mm_var
            self.detail_widgets[f"light_ob_{idx}"] = ob_var

        self.create_button(
            self.form_frame, "➕ Ajouter Light Entrée",
            command=lambda: self.add_portal_entry(p_key, is_light=True),
            row=7 + len(data["light"]), column=0, colspan=4, pady=5
        )

        base_fury_row = 8 + len(data["light"])
        tk.Label(
            self.form_frame, text="fury (entrées)", font=("Arial", 12, "bold"),
            bg="#2e2e2e", fg="white"
        ).grid(row=base_fury_row, column=0, columnspan=2, sticky='w')

        for idx, (m, ob) in enumerate(data["fury"]):
            r = base_fury_row + 1 + idx
            tk.Label(self.form_frame, text=f"{idx+1}. Map :", bg="#2e2e2e", fg="white").grid(
                row=r, column=0, sticky='e'
            )
            mm_var = tk.StringVar(value=str(m))
            self.create_entry(self.form_frame, mm_var, row=r, column=1)
            tk.Label(self.form_frame, text="Obél :", bg="#2e2e2e", fg="white").grid(
                row=r, column=2, sticky='e'
            )
            ob_var = tk.StringVar(value=ob)
            self.create_entry(self.form_frame, ob_var, row=r, column=3)

            self.detail_widgets[f"fury_map_{idx}"] = mm_var
            self.detail_widgets[f"fury_ob_{idx}"] = ob_var

        self.create_button(
            self.form_frame, "➕ Ajouter Fury Entrée",
            command=lambda: self.add_portal_entry(p_key, is_light=False),
            row=base_fury_row + 1 + len(data["fury"]), column=0, colspan=4, pady=5
        )

    def save_portal_header(self, p_key, mapno, pid):
        """Sauvegarde MapNo, PortalID, et reconstruit light/fury."""
        try:
            m = int(mapno)
            p = int(pid)
        except ValueError:
            messagebox.showerror("Erreur", "MapNo et PortalID doivent être des entiers.")
            return

        self.portals[p_key]["mapno"] = m
        self.portals[p_key]["portalid"] = p

        # Reconstruire la liste light
        light_list = []
        idx = 0
        while True:
            mk = f"light_map_{idx}"
            ok = f"light_ob_{idx}"
            if mk not in self.detail_widgets:
                break
            try:
                mm = int(self.detail_widgets[mk].get())
                ob = self.detail_widgets[ok].get().strip()
                light_list.append((mm, ob))
            except ValueError:
                messagebox.showerror("Erreur", f"Entrée light #{idx+1} invalide.")
                return
            idx += 1

        # Reconstruire la liste fury
        fury_list = []
        idx = 0
        while True:
            mk = f"fury_map_{idx}"
            ok = f"fury_ob_{idx}"
            if mk not in self.detail_widgets:
                break
            try:
                mm = int(self.detail_widgets[mk].get())
                ob = self.detail_widgets[ok].get().strip()
                fury_list.append((mm, ob))
            except ValueError:
                messagebox.showerror("Erreur", f"Entrée fury #{idx+1} invalide.")
                return
            idx += 1

        self.portals[p_key]["light"] = light_list
        self.portals[p_key]["fury"] = fury_list
        messagebox.showinfo("Enregistré", f"Portail « {p_key} » mis à jour.")
        self.populate_tree()

    def add_portal_entry(self, p_key, is_light):
        """Ajoute une entrée générique (0, 'Obelisk_001') dans light ou fury."""
        if is_light:
            self.portals[p_key]["light"].append((0, "Obelisk_001"))
        else:
            self.portals[p_key]["fury"].append((0, "Obelisk_001"))

        self.populate_tree()
        # Sélectionner le dernier ajouté dans l'arbre pour faciliter l'édition
        for root_item in self.tree.get_children():
            vals = self.tree.item(root_item, "values")
            if len(vals) >= 1 and vals[0] == "portalroot":
                for p_item in self.tree.get_children(root_item):
                    vals_p = self.tree.item(p_item, "values")
                    if len(vals_p) >= 2 and vals_p[1] == p_key:
                        entries = self.tree.get_children(p_item)
                        self.tree.selection_set(entries[-1])
                        self.tree.see(entries[-1])
                        return

    def show_portal_entry_form(self, p_key, idx, is_light):
        """Formulaire pour éditer une seule entrée light/fury."""
        data = self.portals[p_key]
        lst = data["light"] if is_light else data["fury"]
        m, ob = lst[idx]
        label = "light" if is_light else "fury"
        title = f"{label} Entrée #{idx+1} (Portail {p_key})"
        tk.Label(
            self.form_frame, text=title, font=("Arial", 14, "bold"),
            bg="#2e2e2e", fg="white"
        ).grid(row=0, column=0, columnspan=2, sticky='w')

        self.create_label(self.form_frame, "MapNo :", row=1, column=0)
        map_var = tk.StringVar(value=str(m))
        self.create_entry(self.form_frame, map_var, row=1, column=1)

        self.create_label(self.form_frame, "Obélisque :", row=2, column=0)
        ob_var = tk.StringVar(value=ob)
        self.create_entry(self.form_frame, ob_var, row=2, column=1)

        self.create_button(
            self.form_frame, "ðﾟﾒﾾ Enregistrer Entrée",
            command=lambda: self.save_portal_entry(p_key, idx, map_var.get(), ob_var.get(), is_light),
            row=3, column=0, colspan=2, pady=10
        )
        self.create_button(
            self.form_frame, "❌ Supprimer Entrée",
            command=lambda: self.delete_portal_entry(p_key, idx, is_light),
            row=4, column=0, colspan=2, pady=2
        )

    def save_portal_entry(self, p_key, idx, mapno, ob, is_light):
        """Sauvegarde une entrée light/fury unique."""
        try:
            m = int(mapno)
        except ValueError:
            messagebox.showerror("Erreur", "MapNo doit être un entier.")
            return

        if is_light:
            self.portals[p_key]["light"][idx] = (m, ob)
        else:
            self.portals[p_key]["fury"][idx] = (m, ob)

        messagebox.showinfo("Enregistré", "Entrée mise à jour.")
        self.populate_tree()

    def delete_portal_entry(self, p_key, idx, is_light):
        """Supprime une entrée light ou fury après confirmation."""
        resp = messagebox.askyesno("Confirmation", "Supprimer cette entrée ?")
        if not resp:
            return

        if is_light:
            del self.portals[p_key]["light"][idx]
        else:
            del self.portals[p_key]["fury"][idx]

        messagebox.showinfo("Supprimé", "Entrée supprimée.")
        self.populate_tree()

    def delete_portal(self, p_key):
        """Supprime un portail entier après confirmation."""
        resp = messagebox.askyesno("Confirmation", f"Supprimer le portail « {p_key} » ?")
        if not resp:
            return
        del self.portals[p_key]
        messagebox.showinfo("Supprimé", f"Portail « {p_key} » supprimé.")
        self.populate_tree()

    # ---------- Génération / Sauvegarde du Fichier ----------
    def generate_file_content(self):
        """
        Reconstruit le contenu texte Shaiya à partir des données en mémoire.
        Retourne une chaîne complète.
        """
        lines = []

        # 1) Zones / Obelisks / Mobs
        for zone_key, obelisks in sorted(self.data.items()):
            lines.append(f"[{zone_key}]")
            for ob_key, ob_data in sorted(obelisks.items()):
                x, y, z = ob_data["position"]
                mN, mL, mF = ob_data["mob_ids"]
                fct = ob_data["default_faction"]
                lines.append(f"    [{ob_key}] = ({x:.2f},{y:.2f},{z:.2f}),({mN},{mL},{mF}),{fct}")
                for mob in ob_data["mobs"]:
                    mx, my, mz = mob["position"]
                    mr = mob["radius"]
                    idN, idL, idF = mob["mob_ids"]
                    cnt = mob["count"]
                    lines.append(f"        [Mob] = ({mx:.2f},{my:.2f},{mz:.2f},{mr:.2f}),({idN},{idL},{idF}),{cnt}")
            lines.append("")

        # 2) BothMobs
        for both_key, both_data in sorted(self.bothmobs.items()):
            map_no, change_cnt, respwn, (p_map, p_id) = both_data["params"]
            bx, by, bz = both_data["position"]
            idA, idB = both_data["ids"]
            lines.append(f"[{both_key}, ({map_no},{change_cnt},{respwn},({p_map},{p_id}))]  =  ({bx:.2f},{by:.2f},{bz:.2f}),({idA},{idB})")
            for mob in both_data["mobs"]:
                mx, my, mz = mob["position"]
                mr = mob["radius"]
                idN, idL, idF = mob["mob_ids"]
                cnt = mob["count"]
                lines.append(f"    [Mob]  = ({mx:.3f},{my:.1f},{mz:.2f},{mr:.0f}),({idN},{idL},{idF}),{cnt}")
            if both_data["mobs"]:
                lines.append("")
            if both_data["itemdrops"]:
                lines.append("////////////////////////////////Item Drop///////////////////////////")
                for item in both_data["itemdrops"]:
                    mx, my, mz = item["position"]
                    mr = item["radius"]
                    idN, idL, idF = item["mob_ids"]
                    cnt = item["count"]
                    lines.append(f"    [Mob]  = ({mx:.0f},{my:.0f},{mz:.0f},{mr:.0f}),({idN},{idL},{idF}),{cnt}")
                lines.append("")

        # 3) Portails
        for p_key, p_data in sorted(self.portals.items()):
            lines.append(f"[{p_key}, ({p_data['mapno']},{p_data['portalid']}) ]")
            if p_data["light"]:
                light_entries = ",".join(f"({m},{ob})" for (m, ob) in p_data["light"])
                lines.append(f"[light] = {light_entries}")
            if p_data["fury"]:
                fury_entries = ",".join(f"({m},{ob})" for (m, ob) in p_data["fury"])
                lines.append(f"[fury]  = {fury_entries}")
            lines.append("")

        return "\n".join(lines).strip() + "\n"

    def save_file(self):
        """Sauvegarde dans le fichier chargé."""
        if not hasattr(self, 'current_file'):
            messagebox.showwarning("Aucun fichier", "Aucun fichier chargé.")
            return
        content = self.generate_file_content()
        try:
            with open(self.current_file, 'w', encoding='utf-8') as f:
                f.write(content)
            messagebox.showinfo("Enregistré", f"Fichier enregistré :\n{self.current_file}")
        except Exception as e:
            messagebox.showerror("Erreur", f"Impossible d'enregistrer : {e}")

# Intégration dans l'onglet "EditObelisk"
if __name__ == "__main__":
    import tkinter as tk
    from tkinter import ttk

    root = tk.Tk()
    root.title("Mon éditeur multi‐onglets")
    root.geometry("1200x800")  # ajustez selon la taille que vous souhaitez

    # ─── AJOUTER ICI la personnalisation des onglets ───
    style = ttk.Style(root)
    style.theme_use("clam")

    # Couleurs souhaitées (à adapter)
    DARK_FRAME  = "#3e3e3e"   # fond des onglets non-sélectionnés
    DARK_ACTIVE = "#2e2e2e"   # fond de l’onglet actif
    LIGHT_FG    = "#ffffff"   # couleur du texte

    style.configure(
        "TNotebook.Tab",
        background=DARK_FRAME,
        fieldbackground=DARK_FRAME,
        foreground=LIGHT_FG,
        bordercolor=DARK_FRAME,
        borderwidth=0,
        #padding=[5, 5]
    )
    style.map(
        "TNotebook.Tab",
        background=[("selected", "#505050")],   # fond quand l’onglet est actif
        foreground=[("selected", LIGHT_FG)],   # couleur du texte quand actif
        
    )
        # --- NOUVEAU : style de l’en-tête au survol ---


    style.configure("TNotebook", background=DARK_FRAME, borderwidth=0)
    # ────────────────────────────────────────────────────────

    # ─────────────────── Création du notebook ───────────────────
    notebook = ttk.Notebook(root)
    notebook.pack(fill=tk.BOTH, expand=True)

    # ─────────────────── Onglet "EditItemCreate" ───────────────────
    frame_item = IniEditorFrame(notebook)
    notebook.add(frame_item, text="EditItemCreate")

    # ─────────────────── Onglet "EditMobCreate" ───────────────────
    frame_mob = MobCreateFrame(notebook)
    notebook.add(frame_mob, text="EditMobCreate")

    # ─────────────────── Onglet "EditMapCreate" ───────────────────
    frame_map = MapIniEditorFrame(notebook)
    notebook.add(frame_map, text="EditMapCreate")

    # ─────────────────── Onglet "EditObelisk" ───────────────────
    frame_obelisk = ObeliskEditorFrame(notebook)
    notebook.add(frame_obelisk, text="EditObelisk")

    # ─────────── Onglet restant "EditChaoticSquare" ───────────
    # (remplacez ChaoticSquareEditorFrame par le nom de la classe que vous avez définie)
    frame_chaos = ChaoticSquareEditorFrame(notebook)
    notebook.add(frame_chaos, text="EditChaoticSquare")

    # Onglet LuaExplorer
    frame_lua = LuaExplorerFrame(notebook)
    notebook.add(frame_lua, text="EditLua")

    root.mainloop()


