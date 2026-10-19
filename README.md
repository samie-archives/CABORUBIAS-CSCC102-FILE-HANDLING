import io
import csv
import html
import pandas as pd
from IPython.display import display, HTML
import ipywidgets as widgets

# ---------- Widgets ----------
uploader = widgets.FileUpload(
    accept='.csv',
    multiple=False,
    description='Upload CSV',
    button_style='primary'
)

student_dropdown = widgets.Dropdown(
    options=[],
    description='Student:',
    disabled=True,
    layout=widgets.Layout(width='420px')
)

status_label = widgets.HTML()
output_area = widgets.Output()

students = []  # one dict per student: name, course, year, subjects


# ---------- Helpers ----------
def get_uploaded_bytes():
    """Works with both ipywidgets 7 (dict) and 8 (tuple of dicts)."""
    value = uploader.value
    if not value:
        return None
    first = list(value.values())[0] if isinstance(value, dict) else value[0]
    return bytes(first['content'])


def is_number(text):
    try:
        float(text)
        return True
    except (ValueError, TypeError):
        return False


def parse_students(raw_bytes):
    """
    Parses a CSV made of repeated blocks:

        Name, Course, Year              <- student info (label line optional)
        Code, Units, Teacher, Grade     <- subject rows (label line optional)
        ...
    Returns (list_of_students, number_of_orphan_rows).
    """
    try:
        text = raw_bytes.decode('utf-8-sig')
    except UnicodeDecodeError:
        text = raw_bytes.decode('latin-1')

    parsed = []
    current = None
    orphan_rows = 0

    for row in csv.reader(io.StringIO(text)):
        cells = [c.strip() for c in row]
        while cells and cells[-1] == '':
            cells.pop()
        if not cells:
            continue  # blank line

        cells[0] = cells[0].lstrip('*').strip()
        if not cells[0] and len(cells) == 1:
            continue

        # Skip label/header lines
        if cells[0].lower() in ('name', 'code'):
            continue

        # Subject row: 4+ fields, or 2nd field is numeric (Units) under a student
        is_subject = len(cells) >= 4 or (current is not None and len(cells) > 1 and is_number(cells[1]))

        if is_subject:
            if current is None:
                orphan_rows += 1
                continue
            cells += [''] * (4 - len(cells))
            current['subjects'].append({
                'Code': cells[0],
                'Units': pd.to_numeric(cells[1], errors='coerce'),
                'Teacher': cells[2],
                'Grade': pd.to_numeric(cells[3], errors='coerce'),
            })
        else:
            # Student info row
            current = {
                'name': cells[0],
                'course': cells[1] if len(cells) > 1 and cells[1] else 'N/A',
                'year': cells[2] if len(cells) > 2 and cells[2] else 'N/A',
                'subjects': [],
            }
            parsed.append(current)

    return parsed, orphan_rows


def show_report(index):
    with output_area:
        output_area.clear_output(wait=True)

        if index is None or not (0 <= index < len(students)):
            return

        s = students[index]
        df_all = pd.DataFrame(s['subjects'], columns=['Code', 'Units', 'Teacher', 'Grade'])
        df_report = df_all.dropna(subset=['Units', 'Grade']).copy()
        skipped = len(df_all) - len(df_report)

        if df_report.empty:
            print(f"❌ No valid grades found for {s['name']}.")
            return

        total_subjects = len(df_report)
        total_units = df_report['Units'].sum()
        average_grade = df_report['Grade'].mean()

        # General Weighted Average (GWA)
        df_report['Weighted_Grade'] = df_report['Grade'] * df_report['Units']
        gwa = df_report['Weighted_Grade'].sum() / total_units if total_units > 0 else 0

        table_rows = ""
        for _, row in df_report.iterrows():
            table_rows += f"""
            <tr style='border-bottom: 1px solid #E1BEE7;'>
                <td style='padding: 8px; font-size: 13px; color: #4A148C;'>{html.escape(str(row['Code']))}</td>
                <td style='padding: 8px; font-size: 13px; color: #4A148C;'>{html.escape(str(row['Teacher']))}</td>
                <td style='padding: 8px; font-size: 13px; text-align: center; color: #4A148C;'>{row['Units']:g}</td>
                <td style='padding: 8px; font-size: 13px; text-align: center; font-weight: bold; color: #4A148C;'>{row['Grade']:.2f}</td>
            </tr>
            """

        skipped_note = (
            f"<p style='color:#7B1FA2; font-size:12px;'>⚠️ {skipped} row(s) skipped because Units or Grade was not a number.</p>"
            if skipped else ""
        )

        report_html = f"""
        <div id='printable-report' style='font-family: Arial, sans-serif; border: 3px solid #9C27B0; padding: 25px; border-radius: 15px; background-color: #F3E5F5; max-width: 800px; margin: 20px auto; box-shadow: 0 4px 10px rgba(156, 39, 176, 0.15);'>

            <h2 style='color: #4A148C; margin-top: 0; text-align: center; letter-spacing: 2px; font-weight: bold;'>⭐ STUDENT GRADE REPORT ⭐</h2>
            <hr style='border: 1.5px solid #9C27B0;'>

            <table style='width: 100%; font-size: 14px; margin-bottom: 20px;'>
                <tr>
                    <td style='padding: 4px 0; font-weight: bold; width: 25%; color: #4A148C;'>Student Name:</td>
                    <td style='padding: 4px 0; color: #7B1FA2; font-size: 15px; font-weight: bold;'>{html.escape(s['name'])}</td>
                </tr>
                <tr>
                    <td style='padding: 4px 0; font-weight: bold; color: #4A148C;'>Course/Program:</td>
                    <td style='padding: 4px 0; color: #7B1FA2;'>{html.escape(s['course'])}</td>
                </tr>
                <tr>
                    <td style='padding: 4px 0; font-weight: bold; color: #4A148C;'>Year Level:</td>
                    <td style='padding: 4px 0; color: #7B1FA2;'>{html.escape(s['year'])}</td>
                </tr>
            </table>

            <h4 style='color: #6A1B9A; margin-bottom: 10px; border-bottom: 2px solid #E1BEE7; padding-bottom: 5px;'>✨ DETAILED SUBJECT GRADES</h4>

            <table style='width: 100%; border-collapse: collapse; margin-bottom: 25px; background-color: #FFFFFF; border-radius: 8px; overflow: hidden; box-shadow: 0 2px 4px rgba(0,0,0,0.05);'>
                <thead>
                    <tr style='background-color: #D1C4E9; color: #311B92;'>
                        <th style='padding: 10px; text-align: left;'>Subject Code</th>
                        <th style='padding: 10px; text-align: left;'>Teacher</th>
                        <th style='padding: 10px; text-align: center;'>Units</th>
                        <th style='padding: 10px; text-align: center;'>Grade</th>
                    </tr>
                </thead>
                <tbody>
                    {table_rows}
                </tbody>
            </table>
            {skipped_note}

            <h4 style='color: #6A1B9A; margin-bottom: 10px; border-bottom: 2px solid #E1BEE7; padding-bottom: 5px;'>⭐ GRADE SUMMARY</h4>

            <table style='width: 100%; border-collapse: collapse; font-size: 14px; background-color: #EDE7F6; border-radius: 8px; padding: 10px;'>
                <tr>
                    <td style='padding: 8px; font-weight: bold; color: #4A148C;'>Total Subjects Taken:</td>
                    <td style='padding: 8px; text-align: right; color: #4A148C; font-weight: bold;'>{total_subjects}</td>
                </tr>
                <tr>
                    <td style='padding: 8px; font-weight: bold; color: #4A148C;'>Total Units Earned:</td>
                    <td style='padding: 8px; text-align: right; color: #4A148C; font-weight: bold;'>{total_units:.1f}</td>
                </tr>
                <tr>
                    <td style='padding: 8px; font-weight: bold; color: #4A148C;'>Simple Average Grade:</td>
                    <td style='padding: 8px; text-align: right; color: #4A148C; font-weight: bold;'>{average_grade:.2f}</td>
                </tr>
                <tr style='background-color: #BA68C8; font-size: 16px; border-top: 2px solid #9C27B0;'>
                    <td style='padding: 10px; font-weight: bold; color: #FFFFFF;'>General Weighted Average (GWA):</td>
                    <td style='padding: 10px; text-align: right; font-weight: bold; color: #FFFFFF;'>{gwa:.4f}</td>
                </tr>
            </table>
        </div>

        <div style='text-align: center; margin-top: 15px;'>
            <button onclick="
                var printContents = document.getElementById('printable-report').outerHTML;
                var win = window.open('', '', 'height=1000, width=900');
                win.document.write('<html><head><title>Student Grade Report</title>');
                win.document.write('<style>body {{ margin: 20px; }}</style>');
                win.document.write('</head><body>');
                win.document.write(printContents);
                win.document.write('</body></html>');
                win.document.close();
                win.focus();
                setTimeout(function() {{ win.print(); win.close(); }}, 500);
            " style='background-color: #9C27B0; color: white; border: none; padding: 12px 24px; font-size: 15px; font-weight: bold; border-radius: 8px; cursor: pointer; box-shadow: 0 4px 6px rgba(156, 39, 176, 0.3);'>
                🖨️ Print Report to PDF
            </button>
        </div>
        """
        display(HTML(report_html))


# ---------- Event handlers ----------
def on_student_change(change):
    show_report(change['new'])


def on_upload_change(change):
    global students

    content = get_uploaded_bytes()
    if content is None:
        return

    with output_area:
        output_area.clear_output()

    try:
        students, orphans = parse_students(content)
    except Exception as e:
        status_label.value = f"<span style='color:#C62828;'>❌ Error reading CSV: {html.escape(str(e))}</span>"
        return

    if not students:
        student_dropdown.disabled = True
        status_label.value = "<span style='color:#C62828;'>❌ No students found in the file.</span>"
        return

    # Label by name; add the course if two students share a name
    names = [s['name'] for s in students]
    options = []
    for i, s in enumerate(students):
        label = f"{s['name']} ({s['course']})" if names.count(s['name']) > 1 else s['name']
        options.append((label, i))

    # Swap the options without firing the change handler twice
    student_dropdown.unobserve(on_student_change, names='value')
    student_dropdown.options = options
    student_dropdown.value = 0
    student_dropdown.disabled = False
    student_dropdown.observe(on_student_change, names='value')

    total_subjects = sum(len(s['subjects']) for s in students)
    note = f" ({orphans} subject row(s) before the first student were ignored.)" if orphans else ""
    status_label.value = (
        f"<span style='color:#4A148C;'>✅ Loaded {len(students)} student(s) and {total_subjects} subject row(s).{note}</span>"
    )
    show_report(0)


uploader.observe(on_upload_change, names='value')
student_dropdown.observe(on_student_change, names='value')


# ---------- Layout ----------
input_form = widgets.VBox([
    widgets.HTML("<b style='color: #4A148C; font-size: 16px;'>✨ Student Grade Report Generator ✨</b>"),
    uploader,
    status_label,
    student_dropdown
])

display(input_form)
display(output_area)
