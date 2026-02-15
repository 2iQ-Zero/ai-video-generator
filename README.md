import streamlit as st
from openai import OpenAI
from moviepy.editor import *
from PIL import Image, ImageDraw
import random
import os

# =========================
# 🔐 قراءة المفتاح من Secrets
# =========================
API_KEY = st.secrets["OPENAI_API_KEY"]

client = OpenAI(api_key=API_KEY)

# إعداد الصفحة
st.set_page_config(page_title="مولد الفيديو الذكي", page_icon="🎬")

st.title("🎬 مولد الفيديو الذكي")
st.write("حوّل أي فكرة إلى فيديو احترافي بالذكاء الاصطناعي")

topic = st.text_input("📝 موضوع الفيديو")
style = st.selectbox("🎨 النمط", ["تعليمي", "تحفيزي", "تسويقي", "قصصي"])
duration = st.slider("⏱️ المدة (ثواني)", 15, 60, 30, step=5)

if st.button("🚀 إنشاء الفيديو"):

    if not topic:
        st.warning("اكتب موضوع الأول")
        st.stop()

    try:
        with st.spinner("🤖 جاري كتابة السكريبت..."):

            response = client.chat.completions.create(
                model="gpt-4o-mini",
                messages=[
                    {
                        "role": "system",
                        "content": "أنت كاتب سكريبتات فيديو محترف. اكتب بالعربية الفصحى."
                    },
                    {
                        "role": "user",
                        "content": f"""
                        اكتب سكريبت فيديو {style} عن "{topic}"
                        مدته {duration} ثانية
                        قسمه إلى {duration//5} مشاهد قصيرة
                        """
                    }
                ],
                temperature=0.7
            )

            script = response.choices[0].message.content

        st.success("✅ تم إنشاء السكريبت")
        st.write(script)

        # ====================
        # إنشاء الفيديو
        # ====================

        clips = []
        lines = [l for l in script.split("\n") if l.strip()]

        for i, line in enumerate(lines[:6]):

            img = Image.new("RGB", (1280, 720), color=(30, 60, 120))
            draw = ImageDraw.Draw(img)

            text = line[:80]
            draw.text((640, 360), text, fill="white", anchor="mm")

            img_path = f"scene_{i}.png"
            img.save(img_path)

            clip = ImageClip(img_path).set_duration(5)
            clips.append(clip)

        final = concatenate_videoclips(clips)
        output_file = f"video_{random.randint(1000,9999)}.mp4"
        final.write_videofile(output_file, fps=24, codec="libx264")

        st.video(output_file)

        with open(output_file, "rb") as f:
            st.download_button("📥 تحميل الفيديو", f, file_name=output_file)

        # تنظيف الملفات
        for f in os.listdir():
            if f.startswith("scene_") or f == output_file:
                os.remove(f)

    except Exception as e:
        st.error(f"❌ حصل خطأ: {str(e)}")
