package main

import (
 "fmt"
 "io"
 "log"
 "net/http"
 "os"
 "path/filepath"
 "sync"
 "time"

 "github.com/gin-gonic/gin"
)

const (
 uploadDir   = "./uploads"
 maxFileSize = 1024 * 1024 * 800 // ~800 MB
)

var (
 mu           sync.Mutex
 uploadedFiles = make(map[string]string) // filename → original name
)

func main() {
 // создаём папку для видео, если нет
 if err := os.MkdirAll(uploadDir, 0755); err != nil {
  log.Fatal("не удалось создать папку uploads:", err)
 }

 r := gin.Default()

 // CORS для фронта (можно убрать или настроить жёстче в проде)
 r.Use(func(c *gin.Context) {
  c.Writer.Header().Set("Access-Control-Allow-Origin", "*")
  c.Writer.Header().Set("Access-Control-Allow-Methods", "GET, POST, OPTIONS")
  c.Writer.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")
  if c.Request.Method == "OPTIONS" {
   c.AbortWithStatus(204)
   return
  }
  c.Next()
 })

 r.GET("/", func(c *gin.Context) {
  c.String(200, YouTube-подобный бэкенд (MVP)
  
POST /upload      → загрузить видео (multipart, поле "video")
GET  /videos      → список загруженных видео
GET  /play/:name  → посмотреть видео (прямая ссылка)
  )
 })

 // Загрузка видео
 r.POST("/upload", handleUpload)

 // Список всех загруженных видео
 r.GET("/videos", func(c *gin.Context) {
  mu.Lock()
  defer mu.Unlock()

  var list []string
  for fname := range uploadedFiles {
   list = append(list, fname)
  }

  c.JSON(200, gin.H{
   "videos": list,
   "count":  len(list),
  })
 })

 // Прямая отдача видео (для <video> тега)
 r.GET("/play/:filename", func(c *gin.Context) {
  filename := c.Param("filename")
  path := filepath.Join(uploadDir, filename)

  if _, err := os.Stat(path); os.IsNotExist(err) {
   c.JSON(404, gin.H{"error": "видео не найдено"})
   return
  }

  c.Header("Content-Type", "video/mp4")
  c.Header("Accept-Ranges", "bytes")
  c.File(path)
 })

 log.Println("Сервер запущен → http://localhost:8080")
 log.Fatal(r.Run(":8080"))
}

func handleUpload(c *gin.Context) {
 // ограничиваем размер файла
 c.Request.Body = http.MaxBytesReader(c.Writer, c.Request.Body, maxFileSize)

 file, header, err := c.Request.FormFile("video")
 if err != nil {
  c.JSON(400, gin.H{"error": "поле video не найдено или ошибка чтения"})
  return
 }
 defer file.Close()

 if header.Size > maxFileSize {
  c.JSON(413, gin.H{"error": "файл слишком большой (макс 800 МБ)"})
  return
 }

 // безопасное имя файла
 ext := filepath.Ext(header.Filename)
 if ext != ".mp4" && ext != ".webm" && ext != ".mov" {
  c.JSON(400, gin.H{"error": "поддерживаются только .mp4, .webm, .mov"})
  return
 }

 // уникальное имя
 timestamp := time.Now().Format("20060102-150405")
 safeName := fmt.Sprintf("%s-%s%s", timestamp, header.Filename[:10], ext)
 fullPath := filepath.Join(uploadDir, safeName)

 // сохраняем
 out, err := os.Create(fullPath)
 if err != nil {
  c.JSON(500, gin.H{"error": "не удалось создать файл"})
  return
 }
 defer out.Close()

 _, err = io.Copy(out, file)
 if err != nil {
  c.JSON(500, gin.H{"error": "ошибка при сохранении"})
  return
 }

 // запоминаем
 mu.Lock()
 uploadedFiles[safeName] = header.Filename
 mu.Unlock()

 c.JSON(200, gin.H{
  "message":  "видео загружено",
  "filename": safeName,
  "url":      fmt.Sprintf("http://localhost:8080/play/%s", safeName),
 })
}
