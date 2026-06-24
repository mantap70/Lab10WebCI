Praktikum Pemrograman Web 2

**Nama:** Fathan Atallah Rasya Nugraha
**NIM:** 312410425
**Kelas:** I241C  
**Mata Kuliah:** Pemrograman Web 2  
**Dosen:** Agung Nugroho  
**Universitas:** Pelita Bangsa

---

## Praktikum 5 – Pagination dan Pencarian

### Tujuan
1. Memahami konsep dasar Pagination.
2. Memahami konsep dasar Pencarian.
3. Membuat Paging dan Pencarian menggunakan Framework CodeIgniter 4.

### Membuat Pagination

Pagination digunakan untuk membatasi tampilan data yang panjang menjadi beberapa halaman. CodeIgniter 4 menyediakan library pagination bawaan.

Modifikasi method `admin_index()` pada `app/Controllers/Artikel.php`:
```php
public function admin_index() 
{
    $title = 'Daftar Artikel';
    $model = new ArtikelModel();
    $data  = [
        'title'   => $title,
        'artikel' => $model->paginate(10),
        'pager'   => $model->pager,
    ];
    return view('artikel/admin_index', $data);
}
```

Tambahkan kode berikut di bawah tabel pada `app/Views/artikel/admin_index.php`:
```php
<?= $pager->links(); ?>
```


### Membuat Pencarian

Modifikasi method `admin_index()` untuk menambahkan fitur pencarian:
```php
public function admin_index() 
{
    $title = 'Daftar Artikel';
    $q     = $this->request->getVar('q') ?? '';
    $model = new ArtikelModel();
    $data  = [
        'title'   => $title,
        'q'       => $q,
        'artikel' => $model->like('judul', $q)->paginate(10),
        'pager'   => $model->pager,
    ];
    return view('artikel/admin_index', $data);
}
```

Tambahkan form pencarian sebelum tabel pada view:
```php
<form method="get" class="form-search">
    <input type="text" name="q" value="<?= $q; ?>" placeholder="Cari data">
    <input type="submit" value="Cari" class="btn btn-primary">
</form>
```

Ubah link pager agar mempertahankan parameter pencarian:
```php
<?= $pager->only(['q'])->links(); ?>
```

![📷](img/pencarian.png)

### Kesimpulan

Pada praktikum ini berhasil diimplementasikan fitur pagination menggunakan method `paginate()` bawaan CI4 dan fitur pencarian menggunakan method `like()`. Kombinasi `pager->only()` memastikan parameter pencarian tetap terbawa saat berpindah halaman.

---

## Praktikum 6 – Upload File Gambar

### Tujuan
1. Memahami konsep dasar File Upload.
2. Membuat fitur Upload File menggunakan Framework CodeIgniter 4.

### Upload Gambar pada Artikel

Modifikasi method `add()` pada `app/Controllers/Artikel.php` untuk menangani upload file:
```php
public function add() 
{
    $validation = \Config\Services::validation();
    $validation->setRules(['judul' => 'required']);
    $isDataValid = $validation->withRequest($this->request)->run();

    if ($isDataValid) {
        $file = $this->request->getFile('gambar');
        $file->move(ROOTPATH . 'public/gambar');

        $artikel = new ArtikelModel();
        $artikel->insert([
            'judul'  => $this->request->getPost('judul'),
            'isi'    => $this->request->getPost('isi'),
            'slug'   => url_title($this->request->getPost('judul')),
            'gambar' => $file->getName(),
        ]);
        return redirect('admin/artikel');
    }

    $title = "Tambah Artikel";
    return view('artikel/form_add', compact('title'));
}
```

Tambahkan input file pada `app/Views/artikel/form_add.php`:
```php
<p>
    <input type="file" name="gambar">
</p>
```

Ubah tag form untuk mendukung upload file:
```php
<form action="" method="post" enctype="multipart/form-data">
```

![📷](img/tambahartikel.png)

![📷](img/gambar.png)

![📷](img/immo.png)


### Kesimpulan

Fitur upload gambar berhasil diimplementasikan menggunakan method `getFile()` dan `move()` bawaan CodeIgniter 4. File gambar disimpan di folder `public/gambar` dan nama file disimpan ke database.

---

## Praktikum 7 – Relasi Tabel dan Query Builder

### Tujuan
1. Memahami konsep relasi antar tabel dalam database.
2. Mengimplementasikan relasi One-to-Many.
3. Melakukan query dengan join tabel menggunakan Query Builder.
4. Menampilkan data dari tabel yang berelasi.

### Membuat Tabel Kategori

```sql
CREATE TABLE kategori (
    id_kategori   INT(11) AUTO_INCREMENT,
    nama_kategori VARCHAR(100) NOT NULL,
    slug_kategori VARCHAR(100),
    PRIMARY KEY (id_kategori)
);
```

### Menambahkan Foreign Key pada Tabel Artikel

```sql
ALTER TABLE artikel
ADD COLUMN id_kategori INT(11),
ADD CONSTRAINT fk_kategori_artikel
FOREIGN KEY (id_kategori) REFERENCES kategori(id_kategori);
```

### Membuat Model Kategori

Buat `app/Models/KategoriModel.php`:
```php
<?php
namespace App\Models;

use CodeIgniter\Model;

class KategoriModel extends Model
{
    protected $table            = 'kategori';
    protected $primaryKey       = 'id_kategori';
    protected $useAutoIncrement = true;
    protected $allowedFields    = ['nama_kategori', 'slug_kategori'];
}
```

### Memodifikasi Model Artikel

Tambahkan method `getArtikelDenganKategori()` pada `ArtikelModel.php`:
```php
public function getArtikelDenganKategori()
{
    return $this->db->table('artikel')
        ->select('artikel.*, kategori.nama_kategori')
        ->join('kategori', 'kategori.id_kategori = artikel.id_kategori')
        ->get()
        ->getResultArray();
}
```

### Memodifikasi Controller

Update method `index()` untuk menggunakan join:
```php
public function index()
{
    $title   = 'Daftar Artikel';
    $model   = new ArtikelModel();
    $artikel = $model->getArtikelDenganKategori();
    return view('artikel/index', compact('artikel', 'title'));
}
```

Update method `admin_index()` untuk menambahkan filter kategori:
```php
public function admin_index()
{
    $title       = 'Daftar Artikel (Admin)';
    $model       = new ArtikelModel();
    $q           = $this->request->getVar('q') ?? '';
    $kategori_id = $this->request->getVar('kategori_id') ?? '';

    $builder = $model->table('artikel')
        ->select('artikel.*, kategori.nama_kategori')
        ->join('kategori', 'kategori.id_kategori = artikel.id_kategori');

    if ($q != '')           { $builder->like('artikel.judul', $q); }
    if ($kategori_id != '') { $builder->where('artikel.id_kategori', $kategori_id); }

    $data = [
        'title'       => $title,
        'q'           => $q,
        'kategori_id' => $kategori_id,
        'artikel'     => $builder->paginate(10),
        'pager'       => $model->pager,
        'kategori'    => (new KategoriModel())->findAll(),
    ];

    return view('artikel/admin_index', $data);
}
```

### Memodifikasi View

Update `app/Views/artikel/index.php` untuk menampilkan nama kategori:
```php
<p>Kategori: <?= $row['nama_kategori'] ?></p>
```

Tambahkan dropdown filter kategori pada `admin_index.php`:
```php
<select name="kategori_id" class="form-control mr-2">
    <option value="">Semua Kategori</option>
    <?php foreach ($kategori as $k): ?>
        <option value="<?= $k['id_kategori']; ?>" 
            <?= ($kategori_id == $k['id_kategori']) ? 'selected' : ''; ?>>
            <?= $k['nama_kategori']; ?>
        </option>
    <?php endforeach; ?>
</select>
```

![📷](img/adminartikel.png)

![📷](img/kategori.png)

![📷](img/spesifik.png)


### Kesimpulan

Pada praktikum ini berhasil diimplementasikan relasi One-to-Many antara tabel `artikel` dan `kategori` menggunakan Query Builder CI4. Fitur filter berdasarkan kategori ditambahkan pada halaman admin, dan nama kategori berhasil ditampilkan pada halaman publik.

---

## Praktikum 8 – AJAX

### Tujuan
1. Memahami konsep AJAX dan cara kerjanya.
2. Mengimplementasikan AJAX pada aplikasi web dengan CodeIgniter 4.
3. Melatih kemampuan problem solving dan debugging.

### Apa itu AJAX?

AJAX (Asynchronous JavaScript and XML) adalah teknik pengembangan web yang memungkinkan aplikasi web memperbarui data dari server **tanpa harus melakukan reload halaman secara keseluruhan**, sehingga aplikasi terasa lebih responsif.

### Cara Kerja AJAX

1. Pengguna melakukan aksi (klik tombol, isi form, dsb.).
2. JavaScript membuat HTTP request ke server (GET/POST).
3. Server memproses dan mengembalikan response (biasanya JSON).
4. JavaScript memperbarui bagian halaman tanpa reload.

### Menambahkan Library jQuery

Download jQuery dari `https://jquery.com`, lalu salin `jquery-3.6.0.min.js` ke folder `public/assets/js/`.

### Membuat AJAX Controller

Buat `app/Controllers/AjaxController.php`:
```php
<?php
namespace App\Controllers;

use App\Models\ArtikelModel;

class AjaxController extends Controller
{
    public function index()
    {
        return view('ajax/index');
    }

    public function getData()
    {
        $model = new ArtikelModel();
        $data  = $model->findAll();
        return $this->response->setJSON($data);
    }

    public function delete($id)
    {
        $model = new ArtikelModel();
        $model->delete($id);
        return $this->response->setJSON(['status' => 'OK']);
    }
}
```

### Membuat View AJAX

Buat `app/Views/ajax/index.php`:
```php
<?= $this->include('template/header'); ?>
<h1>Data Artikel</h1>
<table class="table-data" id="artikelTable">
    <thead>
        <tr>
            <th>ID</th><th>Judul</th><th>Status</th><th>Aksi</th>
        </tr>
    </thead>
    <tbody></tbody>
</table>
<script src="<?= base_url('assets/js/jquery-3.6.0.min.js') ?>"></script>
<script>
$(document).ready(function() {
    function loadData() {
        $('#artikelTable tbody').html('<tr><td colspan="4">Loading data...</td></tr>');
        $.ajax({
            url: "<?= base_url('ajax/getData') ?>",
            method: "GET",
            dataType: "json",
            success: function(data) {
                var tableBody = "";
                for (var i = 0; i < data.length; i++) {
                    var row = data[i];
                    tableBody += '<tr>';
                    tableBody += '<td>' + row.id + '</td>';
                    tableBody += '<td>' + row.judul + '</td>';
                    tableBody += '<td>---</td>';
                    tableBody += '<td>';
                    tableBody += '<a href="<?= base_url('artikel/edit/') ?>' + row.id + '" class="btn btn-primary">Edit</a>';
                    tableBody += ' <a href="#" class="btn btn-danger btn-delete" data-id="' + row.id + '">Delete</a>';
                    tableBody += '</td>';
                    tableBody += '</tr>';
                }
                $('#artikelTable tbody').html(tableBody);
            }
        });
    }

    loadData();

    $(document).on('click', '.btn-delete', function(e) {
        e.preventDefault();
        var id = $(this).data('id');
        if (confirm('Apakah Anda yakin ingin menghapus artikel ini?')) {
            $.ajax({
                url: "<?= base_url('ajax/delete/') ?>" + id,
                method: "DELETE",
                success: function(data) { loadData(); },
                error: function(jqXHR, textStatus, errorThrown) {
                    alert('Error: ' + textStatus + errorThrown);
                }
            });
        }
    });
});
</script>
<?= $this->include('template/footer'); ?>
```

### Kesimpulan

AJAX memungkinkan komunikasi asinkronus antara browser dan server. Dengan jQuery `$.ajax()`, data dapat dimuat, ditampilkan, dan dihapus dari tabel tanpa perlu me-reload halaman, meningkatkan pengalaman pengguna secara signifikan.

---

## Praktikum 9 – AJAX Pagination dan Search

### Tujuan
1. Memahami konsep AJAX untuk pagination dan search.
2. Mengimplementasikan pagination dan search menggunakan AJAX dalam CodeIgniter 4.
3. Meningkatkan performa dan User Experience (UX) aplikasi web.

### Modifikasi Controller Artikel

Update method `admin_index()` agar mendukung request AJAX:
```php
public function admin_index()
{
    $title       = 'Daftar Artikel (Admin)';
    $model       = new ArtikelModel();
    $q           = $this->request->getVar('q') ?? '';
    $kategori_id = $this->request->getVar('kategori_id') ?? '';
    $page        = $this->request->getVar('page') ?? 1;

    $builder = $model->table('artikel')
        ->select('artikel.*, kategori.nama_kategori')
        ->join('kategori', 'kategori.id_kategori = artikel.id_kategori');

    if ($q != '')           { $builder->like('artikel.judul', $q); }
    if ($kategori_id != '') { $builder->where('artikel.id_kategori', $kategori_id); }

    $artikel = $builder->paginate(10, 'default', $page);
    $pager   = $model->pager;

    $data = [
        'title'       => $title,
        'q'           => $q,
        'kategori_id' => $kategori_id,
        'artikel'     => $artikel,
        'pager'       => $pager,
    ];

    if ($this->request->isAJAX()) {
        return $this->response->setJSON($data);
    } else {
        $kategoriModel    = new KategoriModel();
        $data['kategori'] = $kategoriModel->findAll();
        return view('artikel/admin_index', $data);
    }
}
```

### Modifikasi View admin_index.php

Hapus tabel statis dan ganti dengan container dinamis, tambahkan jQuery AJAX:
```php
<div id="article-container"></div>
<div id="pagination-container"></div>

<script src="https://code.jquery.com/jquery-3.6.0.min.js"></script>
<script>
$(document).ready(function() {
    const fetchData = (url) => {
        $.ajax({
            url: url, type: 'GET', dataType: 'json',
            headers: { 'X-Requested-With': 'XMLHttpRequest' },
            success: function(data) {
                renderArticles(data.artikel);
                renderPagination(data.pager, data.q, data.kategori_id);
            }
        });
    };

    const renderArticles = (articles) => {
        let html = '<table class="table"><thead><tr><th>ID</th><th>Judul</th><th>Kategori</th><th>Status</th><th>Aksi</th></tr></thead><tbody>';
        if (articles.length > 0) {
            articles.forEach(article => {
                html += `<tr>
                    <td>${article.id}</td>
                    <td><b>${article.judul}</b><p><small>${article.isi.substring(0, 50)}</small></p></td>
                    <td>${article.nama_kategori}</td>
                    <td>${article.status}</td>
                    <td>
                        <a class="btn btn-sm btn-info" href="/admin/artikel/edit/${article.id}">Ubah</a>
                        <a class="btn btn-sm btn-danger" onclick="return confirm('Yakin menghapus data?');" href="/admin/artikel/delete/${article.id}">Hapus</a>
                    </td>
                </tr>`;
            });
        } else {
            html += '<tr><td colspan="5">Tidak ada data.</td></tr>';
        }
        html += '</tbody></table>';
        $('#article-container').html(html);
    };

    const renderPagination = (pager, q, kategori_id) => {
        let html = '<nav><ul class="pagination">';
        pager.links.forEach(link => {
            let url = link.url ? `${link.url}&q=${q}&kategori_id=${kategori_id}` : '#';
            html += `<li class="page-item ${link.active ? 'active' : ''}"><a class="page-link" href="${url}">${link.title}</a></li>`;
        });
        html += '</ul></nav>';
        $('#pagination-container').html(html);
    };

    $('#search-form').on('submit', function(e) {
        e.preventDefault();
        fetchData(`/admin/artikel?q=${$('#search-box').val()}&kategori_id=${$('#category-filter').val()}`);
    });

    $('#category-filter').on('change', function() { $('#search-form').trigger('submit'); });

    fetchData('/admin/artikel');
});
</script>
```

![📷](img/adminartikel.png)


### Kesimpulan

Implementasi AJAX pada pagination dan pencarian meningkatkan User Experience karena data diperbarui secara dinamis. Server mendeteksi request AJAX menggunakan `isAJAX()` dan merespons dengan JSON, sementara browser merender ulang hanya bagian tabel dan pagination.

---

## Praktikum 10 – REST API

### Tujuan
1. Memahami konsep dasar API dan RESTful.
2. Membuat API menggunakan Framework CodeIgniter 4.

### Apa itu REST API?

REST (Representational State Transfer) adalah arsitektur API yang menggunakan prinsip HTTP untuk komunikasi data. Data yang dikirim/diterima biasanya berformat **JSON**, sehingga mudah diintegrasikan lintas platform dan bahasa pemrograman.

### Persiapan

Download dan install **Postman** dari `https://www.postman.com/downloads/` untuk pengujian REST API.

### Membuat REST Controller

Buat `app/Controllers/Post.php`:
```php
<?php
namespace App\Controllers;

use CodeIgniter\RESTful\ResourceController;
use CodeIgniter\API\ResponseTrait;
use App\Models\ArtikelModel;

class Post extends ResourceController
{
    use ResponseTrait;

    // GET semua data
    public function index()
    {
        $model          = new ArtikelModel();
        $data['artikel'] = $model->orderBy('id', 'DESC')->findAll();
        return $this->respond($data);
    }

    // POST tambah data
    public function create()
    {
        $model = new ArtikelModel();
        $data  = [
            'judul' => $this->request->getVar('judul'),
            'isi'   => $this->request->getVar('isi'),
        ];
        $model->insert($data);
        return $this->respondCreated([
            'status' => 201, 'error' => null,
            'messages' => ['success' => 'Data artikel berhasil ditambahkan.']
        ]);
    }

    // GET data spesifik
    public function show($id = null)
    {
        $model = new ArtikelModel();
        $data  = $model->where('id', $id)->first();
        if ($data) {
            return $this->respond($data);
        }
        return $this->failNotFound('Data tidak ditemukan.');
    }

    // PUT update data
    public function update($id = null)
    {
        $model = new ArtikelModel();
        $data  = [
            'judul' => $this->request->getVar('judul'),
            'isi'   => $this->request->getVar('isi'),
        ];
        $model->update($id, $data);
        return $this->respond([
            'status' => 200, 'error' => null,
            'messages' => ['success' => 'Data artikel berhasil diubah.']
        ]);
    }

    // DELETE hapus data
    public function delete($id = null)
    {
        $model = new ArtikelModel();
        if ($model->where('id', $id)->first()) {
            $model->delete($id);
            return $this->respondDeleted([
                'status' => 200, 'error' => null,
                'messages' => ['success' => 'Data artikel berhasil dihapus.']
            ]);
        }
        return $this->failNotFound('Data tidak ditemukan.');
    }
}
```

### Membuat Routing REST API

Tambahkan pada `app/Config/Routes.php`:
```php
$routes->resource('post');
```

Cek endpoint yang dihasilkan:
```bash
php spark routes
```

Satu baris `resource('post')` menghasilkan banyak endpoint (GET, POST, PUT, PATCH, DELETE).

### Testing dengan Postman

| Method | URL | Fungsi |
|--------|-----|--------|
| GET | `http://localhost:8080/post` | Tampilkan semua data |
| GET | `http://localhost:8080/post/2` | Tampilkan data ID 2 |
| POST | `http://localhost:8080/post` | Tambah data baru |
| PUT | `http://localhost:8080/post/2` | Ubah data ID 2 |
| DELETE | `http://localhost:8080/post/7` | Hapus data ID 7 |

![📷](img/routes.png)

### Kesimpulan

REST API berhasil dibuat menggunakan `ResourceController` CI4. Dengan satu baris `$routes->resource('post')`, semua endpoint CRUD otomatis terdaftar. Pengujian menggunakan Postman membuktikan semua method (GET, POST, PUT, DELETE) berjalan dengan baik.

---

## Praktikum 11 – VueJS

### Tujuan
1. Memahami konsep dasar API.
2. Memahami konsep dasar Framework VueJS.
3. Membuat Frontend API menggunakan Framework VueJS 3.

### Apa itu VueJS?

VueJS adalah framework JavaScript untuk membangun antarmuka web yang interaktif. Fitur utamanya meliputi reactive data binding, component-based architecture, dan mudah diintegrasikan dengan REST API backend.

### Persiapan

Buat folder `lab8_vuejs` di `htdocs` dengan struktur berikut:
```
│ index.html
└───assets
    ├───css
    │   style.css
    └───js
        app.js
```

Library via CDN:
```html
<script src="https://unpkg.com/vue@3/dist/vue.global.js"></script>
<script src="https://unpkg.com/axios/dist/axios.min.js"></script>
```

### Menampilkan Data (index.html)

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Frontend Vuejs</title>
    <script src="https://unpkg.com/vue@3/dist/vue.global.js"></script>
    <script src="https://unpkg.com/axios/dist/axios.min.js"></script>
    <link rel="stylesheet" href="assets/css/style.css">
</head>
<body>
    <div id="app">
        <h1>Daftar Artikel</h1>
        <table>
            <thead>
                <tr><th>ID</th><th>Judul</th><th>Status</th><th>Aksi</th></tr>
            </thead>
            <tbody>
                <tr v-for="(row, index) in artikel">
                    <td class="center-text">{{ row.id }}</td>
                    <td>{{ row.judul }}</td>
                    <td>{{ statusText(row.status) }}</td>
                    <td class="center-text">
                        <a href="#" @click="edit(row)">Edit</a>
                        <a href="#" @click="hapus(index, row.id)">Hapus</a>
                    </td>
                </tr>
            </tbody>
        </table>
    </div>
    <script src="assets/js/app.js"></script>
</body>
</html>
```

### File app.js (Lengkap dengan CRUD)

```javascript
const { createApp } = Vue
const apiUrl = 'http://localhost/labci4/public'

createApp({
    data() {
        return {
            artikel: '',
            formData: { id: null, judul: '', isi: '', status: 0 },
            showForm: false,
            formTitle: 'Tambah Data',
            statusOptions: [
                {text: 'Draft', value: 0},
                {text: 'Publish', value: 1},
            ],
        }
    },
    mounted() { this.loadData() },
    methods: {
        loadData() {
            axios.get(apiUrl + '/post')
                .then(response => { this.artikel = response.data.artikel })
                .catch(error => console.log(error))
        },
        tambah() {
            this.showForm  = true
            this.formTitle = 'Tambah Data'
            this.formData  = { id: null, judul: '', isi: '', status: 0 }
        },
        edit(data) {
            this.showForm  = true
            this.formTitle = 'Ubah Data'
            this.formData  = { id: data.id, judul: data.judul, isi: data.isi, status: data.status }
        },
        hapus(index, id) {
            if (confirm('Yakin menghapus data?')) {
                axios.delete(apiUrl + '/post/' + id)
                    .then(() => { this.artikel.splice(index, 1) })
                    .catch(error => console.log(error))
            }
        },
        saveData() {
            if (this.formData.id) {
                axios.put(apiUrl + '/post/' + this.formData.id, this.formData)
                    .then(() => { this.loadData() })
                    .catch(error => console.log(error))
            } else {
                axios.post(apiUrl + '/post', this.formData)
                    .then(() => { this.loadData() })
                    .catch(error => console.log(error))
            }
            this.formData  = { id: null, judul: '', isi: '', status: 0 }
            this.showForm  = false
        },
        statusText(status) {
            if (!status) return ''
            return status == 1 ? 'Publish' : 'Draft'
        }
    },
}).mount('#app')
```

![📷](img/vuejs.png)

![📷](img/edit.png)

![📷](img/hapus.png)

### Kesimpulan

VueJS berhasil diintegrasikan dengan REST API CodeIgniter 4 menggunakan Axios untuk operasi CRUD. Reaktivitas Vue membuat tampilan tabel otomatis terupdate setiap ada perubahan data tanpa perlu reload halaman.

---

## Praktikum 13 – VueJS Autentikasi & Navigation Guards

### Tujuan
1. Memahami konsep keamanan dan pembatasan hak akses rute pada sisi klien.
2. Memahami konsep Navigation Guards (`beforeEach`) pada Vue Router.
3. Membuat API Endpoint autentikasi pada backend CodeIgniter 4.
4. Mengimplementasikan modul Login dan proteksi halaman pada SPA.

### Apa itu Navigation Guards?

Pada arsitektur SPA, seluruh halaman sudah dimuat di browser. Vue Router menyediakan `router.beforeEach()` sebagai interceptor perpindahan rute yang memeriksa status login pengguna sebelum mengizinkan akses ke halaman tertentu.

### Tahap 1: Backend – Auth Controller

Buat `app/Controllers/Api/Auth.php`:
```php
<?php
namespace App\Controllers\Api;

use CodeIgniter\RESTful\ResourceController;
use App\Models\UserModel;

class Auth extends ResourceController
{
    protected $format = 'json';

    public function login()
    {
        $username = $this->request->getVar('username');
        $password = $this->request->getVar('password');
        $model    = new UserModel();
        $user     = $model->where('username', $username)->orWhere('useremail', $username)->first();

        if ($user) {
            if ($password === $user['userpassword'] || password_verify($password, $user['userpassword'])) {
                return $this->respond([
                    'status'   => 200,
                    'error'    => null,
                    'messages' => 'Login Berhasil',
                    'data'     => [
                        'id'       => $user['id'],
                        'username' => $user['username'],
                        'token'    => base64_encode("TOKEN-SECRET-" . $user['username'])
                    ]
                ], 200);
            }
        }
        return $this->failUnauthorized('Username atau Password salah.');
    }
}
```

Tambahkan routing:
```php
$routes->post('api/login', 'Api\Auth::login');
```

### Tahap 2: Frontend – Komponen Login

Struktur direktori frontend yang diperbarui:
```
lab8_vuejs/
│ index.html
└───assets/
    ├───css/
    │   style.css
    └───js/
        │ app.js
        └───components/
            Home.js
            Artikel.js
            Login.js
```

Buat `assets/js/components/Login.js`:
```javascript
const Login = {
    template: `
        <div class="login-container">
            <div class="login-box">
                <h2>Form Login Admin</h2>
                <form @submit.prevent="handleLogin">
                    <div class="form-group">
                        <label>Username / Email</label>
                        <input type="text" v-model="username" placeholder="Masukkan username" required>
                    </div>
                    <div class="form-group">
                        <label>Password</label>
                        <input type="password" v-model="password" placeholder="Masukkan password" required>
                    </div>
                    <button type="submit" class="btn-login">Masuk Aplikasi</button>
                </form>
                <p v-if="errorMessage" class="error-msg">{{ errorMessage }}</p>
            </div>
        </div>
    `,
    data() { return { username: '', password: '', errorMessage: '' } },
    methods: {
        handleLogin() {
            axios.post(apiUrl + '/api/login', { username: this.username, password: this.password })
                .then(response => {
                    if (response.data.status === 200) {
                        localStorage.setItem('isLoggedIn', 'true');
                        localStorage.setItem('userToken', response.data.data.token);
                        this.$router.push('/artikel');
                        window.location.reload();
                    }
                })
                .catch(error => {
                    this.errorMessage = error.response?.data?.messages || 'Terjadi kesalahan jaringan.';
                });
        }
    }
};
```

### Konfigurasi Navigation Guards pada app.js

```javascript
const { createApp } = Vue;
const { createRouter, createWebHashHistory } = VueRouter;
const apiUrl = 'http://localhost/labci4/public';

const routes = [
    { path: '/', component: Home },
    { path: '/login', component: Login },
    { path: '/artikel', component: Artikel, meta: { requiresAuth: true } }
];

const router = createRouter({ history: createWebHashHistory(), routes });

router.beforeEach((to, from, next) => {
    const isAuthenticated = localStorage.getItem('isLoggedIn') === 'true';
    if (to.matched.some(record => record.meta.requiresAuth) && !isAuthenticated) {
        alert('Akses Ditolak! Anda harus login terlebih dahulu.');
        next('/login');
    } else {
        next();
    }
});
```

![📷](img/loginvue.png)

### Kesimpulan

Navigation Guards pada Vue Router mengamankan rute di sisi klien dengan memeriksa status login di `localStorage` sebelum mengizinkan akses. API Login di backend CI4 mengembalikan token yang disimpan di browser untuk keperluan autentikasi selanjutnya.

---

## Praktikum 14 – Keamanan API, Autentikasi Token & Axios Interceptors

### Tujuan
1. Memahami konsep keamanan RESTful API menggunakan Token-Based Authentication.
2. Mengimplementasikan Filters CI4 untuk mengamankan endpoint API.
3. Memahami dan mengimplementasikan Axios Interceptors pada VueJS.
4. Melakukan pengujian transmisi data yang aman secara end-to-end.

### Teori Singkat

Praktikum 13 hanya mengamankan sisi klien (browser). Endpoint API masih bisa ditembak langsung via Postman. Praktikum ini menambahkan **Server-Side Security** menggunakan token.

- Saat login berhasil → server memberikan **Token** unik.
- Token wajib dilampirkan di setiap request pada HTTP Header: `Authorization: Bearer <token>`.
- **Axios Interceptors** mencegat setiap request secara otomatis dan menyuntikkan token dari `localStorage`.

### Tahap 1: Backend – API Auth Filter

Buat `app/Filters/ApiAuthFilter.php`:
```php
<?php
namespace App\Filters;

use CodeIgniter\Filters\FilterInterface;
use CodeIgniter\Http\RequestInterface;
use CodeIgniter\Http\ResponseInterface;
use Config\Services;

class ApiAuthFilter implements FilterInterface
{
    public function before(RequestInterface $request, $arguments = null)
    {
        $authHeader = $request->getServer('HTTP_AUTHORIZATION');
        if (!$authHeader) {
            $response = Services::response();
            $response->setStatusCode(401);
            return $response->setJSON([
                'status'   => 401,
                'error'    => 401,
                'messages' => 'Akses Ditolak. Token tidak ditemukan pada request!'
            ]);
        }

        $token = null;
        if (preg_match('/Bearer\s(\S+)/', $authHeader, $matches)) {
            $token = $matches[1];
        }

        if (!$token || empty($token)) {
            $response = Services::response();
            $response->setStatusCode(401);
            return $response->setJSON([
                'status'   => 401,
                'error'    => 401,
                'messages' => 'Sesi Token tidak valid atau kedaluwarsa!'
            ]);
        }
    }

    public function after(RequestInterface $request, ResponseInterface $response, $arguments = null) {}
}
```

Daftarkan di `app/Config/Filters.php`:
```php
'apiauth' => \App\Filters\ApiAuthFilter::class,
```

Terapkan filter pada route yang dilindungi di `app/Config/Routes.php`:
```php
$routes->post('post',              'Api\Post::create',    ['filter' => 'apiauth']);
$routes->put('post/(:segment)',    'Api\Post::update/$1', ['filter' => 'apiauth']);
$routes->delete('post/(:segment)', 'Api\Post::delete/$1', ['filter' => 'apiauth']);
```

### Tahap 2: Frontend – Axios Interceptors

Tambahkan konfigurasi interceptor di `assets/js/app.js` sebelum inisialisasi Vue:
```javascript
// Interceptor Request — Suntikkan token otomatis
axios.interceptors.request.use(
    (config) => {
        const token = localStorage.getItem('userToken');
        if (token) {
            config.headers['Authorization'] = 'Bearer ' + token;
        }
        return config;
    },
    (error) => { return Promise.reject(error); }
);

// Interceptor Response — Tangkap error 401 secara global
axios.interceptors.response.use(
    (response) => { return response; },
    (error) => {
        if (error.response && error.response.status === 401) {
            alert('Sesi Anda telah berakhir atau Token tidak sah. Silakan login kembali.');
            localStorage.clear();
            window.location.href = '#/login';
            window.location.reload();
        }
        return Promise.reject(error);
    }
);
```
![📷](img/postman.png)

### Kesimpulan & Analisis

| Aspek | Vue Router Navigation Guards | CodeIgniter Filters |
|---|---|---|
| Sisi | Client (Browser) | Server |
| Cara kerja | Memeriksa `localStorage` sebelum render halaman | Memeriksa HTTP Header sebelum proses request |
| Kelemahan | Bisa di-bypass dengan manipulasi `localStorage` | Tidak bisa di-bypass dari sisi klien |
| Fungsi | Mencegah pengguna melihat halaman yang diproteksi | Mencegah manipulasi data langsung ke API |

Kesimpulan: Navigation Guards melindungi **tampilan/UI**, sedangkan CI4 Filters melindungi **data/database**. Keduanya harus digunakan bersamaan untuk keamanan yang menyeluruh (defense in depth).

---

*Laporan ini dibuat sebagai dokumentasi praktikum Pemrograman Web 2, Universitas Pelita Bangsa.*
