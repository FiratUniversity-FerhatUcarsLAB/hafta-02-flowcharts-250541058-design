// Veri yapıları
Course:
    id          // benzersiz ders kimliği
    name
    hours       // dersin haftalık saati (tam sayı veya yarım saat gibi float)
    schedule    // dersin saatleri: liste of TimeSlot

TimeSlot:
    day         // örn: "Mon", "Tue", ... veya 0..6
    start       // başlangıç zamanı (sayı, örn 13.5 => 13:30)
    end         // bitiş zamanı (start < end)

// Öğrencinin kayıt ettiği dersler
StudentSchedule:
    courses     // set veya liste of Course

// Yardımcı fonksiyonlar
function totalHours(schedule: StudentSchedule) -> float:
    sum = 0
    for course in schedule.courses:
        sum = sum + course.hours
    return sum

function countCourses(schedule: StudentSchedule) -> int:
    return length(schedule.courses)

function timesOverlap(ts1: TimeSlot, ts2: TimeSlot) -> bool:
    if ts1.day != ts2.day:
        return false
    // zaman aralığı çakışma kontrolü: (başlangıç1 < bitiş2) ve (başlangıç2 < bitiş1)
    return (ts1.start < ts2.end) and (ts2.start < ts1.end)

function courseConflictsWithSchedule(course: Course, schedule: StudentSchedule) -> bool:
    for existing in schedule.courses:
        for slot1 in course.schedule:
            for slot2 in existing.schedule:
                if timesOverlap(slot1, slot2):
                    return true
    return false

// Ana işlem: dersi ekle
function addCourseToSchedule(course: Course, schedule: StudentSchedule) -> (bool, string):
    // 1) Aynı dersi zaten alıyor mu?
    if course.id in [c.id for c in schedule.courses]:
        return (false, "Ders zaten kayıtlı.")

    // 2) Ders sayısı sınırı
    if countCourses(schedule) + 1 > 8:
        return (false, "En fazla 8 farklı ders alabilirsiniz.")

    // 3) Toplam saat sınırı
    if totalHours(schedule) + course.hours > 30:
        return (false, "Toplam ders saati 30'u geçemez.")

    // 4) Saat çakışması kontrolü
    if courseConflictsWithSchedule(course, schedule):
        return (false, "Ders saatleri mevcut programla çakışıyor.")

    // Tüm kontroller geçti => dersi ekle
    schedule.courses.append(course)
    return (true, "Ders başarıyla eklendi.")

// Dersi silme
function removeCourseFromSchedule(courseId: idType, schedule: StudentSchedule) -> (bool, string):
    for i, c in enumerate(schedule.courses):
        if c.id == courseId:
            remove schedule.courses[i]
            return (true, "Ders silindi.")
    return (false, "Ders bulunamadı.")

// Örnek: programı doğrulama (tüm kuralları kontrol et)
function validateFullSchedule(schedule: StudentSchedule) -> (bool, list):
    errors = []
    if countCourses(schedule) > 8:
        errors.append("Ders sayısı 8'i aşıyor.")
    if totalHours(schedule) > 30:
        errors.append("Toplam saat 30'u aşıyor.")
    // çakışma var mı?
    for i from 0 to length(schedule.courses)-1:
        for j from i+1 to length(schedule.courses)-1:
            if courseConflictsWithSchedule(schedule.courses[i], StudentSchedule with courses=[schedule.courses[j]]):
                errors.append("Çakışma: " + schedule.courses[i].id + " ile " + schedule.courses[j].id)
    if errors is empty:
        return (true, [])
    else:
        return (false, errors)

// Örnek kullanım
// dersleri oluştur (basit örnek)
courseA = Course(id="MATH101", name="Calculus", hours=4, schedule=[TimeSlot("Mon", 9, 11), TimeSlot("Wed", 9, 11)])
courseB = Course(id="CS101", name="Intro CS", hours=6, schedule=[TimeSlot("Mon", 10, 12)])  // çakışır
courseC = Course(id="HIST10", name="History", hours=3, schedule=[TimeSlot("Tue", 14, 16)])

student = StudentSchedule(courses=[])

// eklemeye çalış
print addCourseToSchedule(courseA, student) // başarılı
print addCourseToSchedule(courseB, student) // hata: çakışma (Mon 10-12 ile Mon 9-11 kesişiyor)
print addCourseToSchedule(courseC, student) // başarılı
