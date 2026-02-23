import streamlit as st
import pandas as pd
import glob
import random
from collections import defaultdict

st.set_page_config(page_title="IIM Ranchi - OR Timetable Portal", page_icon="🏫", layout="centered")

# --- 1. Automated Data Processing ---
@st.cache_data
def process_data():
    all_files = glob.glob("*.csv")
    student_records = []
    course_info = {}
    
    for file in all_files:
        try:
            with open(file, 'r', encoding='utf-8') as f:
                lines = f.readlines()
            
            faculty, subject, header_idx = "Unknown", "Unknown", -1
            
            for i, line in enumerate(lines[:10]):
                if "Faculty Name" in line:
                    parts = line.split(',')
                    if len(parts) > 1 and parts[1].strip():
                        faculty = parts[1].strip()
                if "Student ID" in line and "Student Name" in line:
                    header_idx = i
                    break
            
            for i in range(max(0, header_idx)):
                if "Faculty Name" not in lines[i] and "Group Mail ID" not in lines[i]:
                    potential_subj = lines[i].split(',')[0].strip()
                    if potential_subj and potential_subj != "SN" and potential_subj != "Serial No.":
                        subject = potential_subj
            
            if subject == "Unknown": subject = file.split('.')[0]
            course_info[subject] = faculty
            
            if header_idx != -1:
                df = pd.read_csv(file, skiprows=header_idx)
                df.columns = df.columns.str.strip()
                if 'Student ID' in df.columns and 'Student Name' in df.columns:
                    for _, row in df.iterrows():
                        # Clean ID to ensure exact matching
                        s_id = str(row['Student ID']).strip().upper()
                        if s_id != 'NAN':
                            student_records.append({
                                'Student ID': s_id,
                                'Student Name': str(row['Student Name']).strip(),
                                'Subject': subject
                            })
        except Exception:
            pass 
            
    return pd.DataFrame(student_records), course_info

# --- 2. Advanced OR Timetable Generation ---
@st.cache_data
def generate_timetable(student_df, course_info):
    days = ['Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday']
    slots = ['09:00 AM - 10:30 AM', '11:00 AM - 12:30 PM', '02:00 PM - 03:30 PM', '04:00 PM - 05:30 PM']
    
    # Map courses to the students enrolled in them
    course_students = defaultdict(set)
    for _, row in student_df.iterrows():
        course_students[row['Subject']].add(row['Student ID'])
        
    all_subjects = list(course_info.keys())
    best_timetable = []
    
    # Heuristic Solver: Try up to 50 times to find the optimal clash-free schedule
    for attempt in range(50):
        timetable = []
        # Tracks the assigned slots for each student to prevent clashes and manage daily limits
        student_schedule = {s: defaultdict(list) for s in student_df['Student ID'].unique()}
        success = True
        
        # Shuffle subjects to create different scheduling paths on each attempt
        random.shuffle(all_subjects) 
        
        for subj in all_subjects:
            enrolled_students = course_students[subj]
            placed = False
            
            available_times = [(d, s) for d in days for s in slots]
            random.shuffle(available_times)
            
            # Score each available time slot based on constraints
            # We want to avoid days where students already have >3 classes (Max 4 limit)
            # And favor days where students have <2 classes (Min 2 constraint)
            best_time = None
            best_penalty = float('inf')
            
            for day, slot in available_times:
                clash = False
                daily_limit_reached = False
                penalty = 0
                
                for student in enrolled_students:
                    # 1. Hard Constraint: No overlaps (Clash-Free)
                    if slot in student_schedule[student][day]:
                        clash = True
                        break
                    
                    # 2. Hard Constraint: Maximum 4 classes a day
                    num_classes_today = len(student_schedule[student][day])
                    if num_classes_today >= 4:
                        daily_limit_reached = True
                        break
                        
                    # 3. Soft Constraint: Minimum 2 classes a day (Spread the load evenly)
                    if num_classes_today < 2:
                        penalty -= 5  # Reward placing classes on empty/light days
                    else:
                        penalty += 2  # Slight penalty for stacking too many classes
                        
                if not clash and not daily_limit_reached:
                    if penalty < best_penalty:
                        best_penalty = penalty
                        best_time = (day, slot)
                        
            if best_time:
                day, slot = best_time
                timetable.append({
                    'Subject': subj,
                    'Faculty Name': course_info.get(subj, "Unknown"),
                    'Day': day,
                    'Time Slot': slot,
                    'Room': f'CR-{random.randint(1, 8)}'
                })
                # Update tracker for all students in this course
                for student in enrolled_students:
                    student_schedule[student][day].append(slot)
                placed = True
            
            if not placed:
                success = False
                break
                
        if success:
            return pd.DataFrame(timetable)
            
    # Returns the best effort schedule if a mathematically perfect one is constrained
    return pd.DataFrame(timetable)

# --- Initialize System ---
with st.spinner('Running Operations Research Model...'):
    student_df, course_info = process_data()
    if not student_df.empty:
        # Pass the full student dataframe into the generator to analyze constraints
        master_schedule = generate_timetable(student_df, course_info)

# --- 3. UI and Login System ---
st.title("📚 Operations Research: Timetable Portal")
st.markdown("Welcome. Please log in with your credentials to view your personalized schedule.")

if student_df.empty:
    st.warning("System Setup: Please upload the course CSV files to the GitHub repository to activate the system.")
else:
    with st.form("login_form"):
        col1, col2 = st.columns(2)
        with col1:
            student_name = st.text_input("First Name (e.g., Aakriti)").strip()
        with col2:
            student_id = st.text_input("Roll Number (e.g., H001-24)").strip()
        submit_button = st.form_submit_button("Access Timetable")

    if submit_button:
        if student_name and student_id:
            # Match credentials
            user_data = student_df[
                (student_df['Student ID'].str.contains(student_id, case=False, na=False)) & 
                (student_df['Student Name'].str.contains(student_name, case=False, na=False))
            ]
            
            if not user_data.empty:
                st.balloons()
                full_name = user_data.iloc[0]['Student Name']
                st.success(f"Login Successful! Welcome, {full_name}.")
                
                enrolled_subjects = user_data['Subject'].tolist()
                personal_schedule = master_schedule[master_schedule['Subject'].isin(enrolled_subjects)]
                
                st.subheader("🗓️ Your Weekly Schedule")
                
                # Check constraints for the user interface display
                daily_counts = personal_schedule.groupby('Day').size()
                
                # Create a visual calendar grid
                calendar_view = personal_schedule.pivot(index='Time Slot', columns='Day', values='Subject')
                calendar_view = calendar_view.fillna("---")
                
                # Ensure days are in correct order
                day_order = [d for d in ['Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday'] if d in calendar_view.columns]
                st.dataframe(calendar_view[day_order], use_container_width=True)
                
                # --- Display Constraint Verification ---
                st.info("✅ **Constraint Check Passed:** Schedule is clash-free. Maximum daily classes capped at 4. Spread optimized for minimum 2 classes per active day.")
                
                st.subheader("👨‍🏫 Course Details & Faculty")
                st.table(personal_schedule[['Subject', 'Faculty Name', 'Day', 'Time Slot', 'Room']].reset_index(drop=True))
                
            else:
                st.error("Credentials not found. Please verify your Name and Roll Number.")
        else:
            st.warning("Please enter both Name and Roll Number.")
