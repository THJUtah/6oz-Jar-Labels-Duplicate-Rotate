import io
import math
import streamlit as st
import fitz  # PyMuPDF

# --- Constants ---
MM_TO_PT = 72.0 / 25.4  # 1 mm = 2.834645669... pt
DEFAULT_GAP_MM = 3.0

st.set_page_config(page_title="Rotate & Duplicate (3 mm gap)", page_icon="🧭", layout="centered")

st.title("Rotate 90° CCW, Duplicate Side-by-Side with 3 mm Gap")
st.caption("Upload a PDF or image (PNG/JPG). Each page is rotated 90° counter-clockwise, duplicated horizontally with an exact 3 mm gap, and exported as a print-ready PDF (0 mm margins).")

with st.sidebar:
    st.header("Settings")
    gap_mm = st.number_input("Gap between copies (mm)", min_value=0.0, step=0.5, value=DEFAULT_GAP_MM, help="Exact visual gap between the two copies on each output page.")
    rotate_ccw = True  # fixed per your spec
    st.write("Rotation: **90° CCW**")
    st.write("Margins: **0 mm**")
    st.write("Layout: **Horizontal (side-by-side)**")
    st.write("Page sizing: **Auto to fit**")

uploaded = st.file_uploader(
    "Upload PDF or image (PDF, PNG, JPG/JPEG)",
    type=["pdf", "png", "jpg", "jpeg"],
    accept_multiple_files=False
)

def _image_file_to_pdf_bytes(file_bytes: bytes, filetype: str) -> bytes:
    """
    Convert a single image (PNG/JPG) to a single-page PDF, preserving pixel
    dimensions as points at 72 DPI (1 px = 1 pt). This is standard for PDF embedding.
    """
    img_doc = fitz.open(stream=file_bytes, filetype=filetype)
    # For safety, take first image only if multi-image formats were used
    page = img_doc[0]
    # Wrap image page content into a PDF
    pdf_bytes = img_doc.convert_to_pdf()
    img_doc.close()
    return pdf_bytes

def _open_as_pdf(file_bytes: bytes, filename: str) -> fitz.Document:
    """
    Open uploaded content as a PDF fitz.Document.
    - If it's already a PDF, open directly.
    - If it's PNG/JPG, convert to a one-page PDF then open.
    """
    name_lower = filename.lower()
    if name_lower.endswith(".pdf"):
        return fitz.open(stream=file_bytes, filetype="pdf")
    elif name_lower.endswith((".png", ".jpg", ".jpeg")):
        filetype = "png" if name_lower.endswith(".png") else "jpg"
        pdf_bytes = _image_file_to_pdf_bytes(file_bytes, filetype=filetype)
        return fitz.open(stream=pdf_bytes, filetype="pdf")
    else:
        # Fallback: attempt PDF first, else image-jpg
        try:
            return fitz.open(stream=file_bytes, filetype="pdf")
        except Exception:
            pdf_bytes = _image_file_to_pdf_bytes(file_bytes, filetype="jpg")
            return fitz.open(stream=pdf_bytes, filetype="pdf")

def process_document_to_two_up(src_doc: fitz.Document, gap_mm: float) -> bytes:
    """
    For each page in src_doc:
      - rotate content 90° CCW
      - duplicate side-by-side with a precise gap_mm between them
      - auto-size the new page to fit both copies exactly (0 margins)
    Return: bytes of the output PDF.
    """
    out_doc = fitz.open()
    gap_pt = gap_mm * MM_TO_PT

    for pno in range(len(src_doc)):
        src_page = src_doc[pno]
        # Source page size in points (width x height)
        sw, sh = src_page.rect.width, src_page.rect.height

        # After 90° CCW rotation, the "visual" width = sh, height = sw
        rot_w, rot_h = sh, sw

        # Output page size:
        # width = two copies of rot_w + gap; height = rot_h
        out_w = (rot_w * 2.0) + gap_pt
        out_h = rot_h

        # Create the output page with exact size
        out_page = out_doc.new_page(width=out_w, height=out_h)

        # Define destination rectangles for the two copies
        # Copy 1 rect: from x=0 to x=rot_w
        left_rect = fitz.Rect(0, 0, rot_w, rot_h)
        # Copy 2 rect: starts after rot_w + gap
        right_rect = fitz.Rect(rot_w + gap_pt, 0, (rot_w * 2.0) + gap_pt, rot_h)

        # Place the original page content into each rect rotated 90° CCW.
        # show_pdf_page will scale the placed page content to the target rect.
        # rotate value is degrees CCW.
        out_page.show_pdf_page(left_rect, src_doc, pno, rotate=90)
        out_page.show_pdf_page(right_rect, src_doc, pno, rotate=90)

    # Export to bytes
    pdf_bytes = out_doc.tobytes()
    out_doc.close()
    return pdf_bytes

if uploaded is not None:
    try:
        file_bytes = uploaded.read()
        src_pdf = _open_as_pdf(file_bytes, uploaded.name)

        # Quick info panel
        with st.expander("Input details", expanded=False):
            st.write(f"Pages detected: **{len(src_pdf)}**")
            if len(src_pdf) > 0:
                w, h = src_pdf[0].rect.width, src_pdf[0].rect.height
                st.write(f"First page size (pt): **{w:.2f} × {h:.2f}**")
                st.write(f"First page size (in): **{w/72.0:.3f} × {h/72.0:.3f}**")

        if st.button("Process & Generate PDF", type="primary"):
            out_pdf_bytes = process_document_to_two_up(src_pdf, gap_mm=gap_mm)
            src_pdf.close()

            # Provide download
            base_name = uploaded.name.rsplit(".", 1)[0]
            out_name = f"{base_name}_rotCCW_two-up_gap{int(round(gap_mm))}mm.pdf"

            st.success("Done! Your print-ready PDF is ready.")
            st.download_button(
                label="⬇️ Download Output PDF",
                data=out_pdf_bytes,
                file_name=out_name,
                mime="application/pdf"
            )

            # Preview first page as an image (optional)
            try:
                preview_doc = fitz.open(stream=out_pdf_bytes, filetype="pdf")
                pix = preview_doc[0].get_pixmap(matrix=fitz.Matrix(150/72, 150/72))  # ~150 DPI preview
                st.image(pix.tobytes("png"), caption="First output page preview", use_container_width=True)
                preview_doc.close()
            except Exception:
                st.info("Preview not available, but the download is ready.")

    except Exception as e:
        st.error(f"Could not process file. {e}")
