<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>OpenSphere Nexus</title>
  <link rel="manifest" href="manifest.json">
  <meta name="apple-mobile-web-app-capable" content="yes">
  <meta name="apple-mobile-web-app-status-bar-style" content="black">
  <style>
    body {font-family:Arial,sans-serif;margin:0;background:#121212;color:#fff;}
    header{background:#1f1f1f;padding:15px;text-align:center;font-size:1.5em;}
    main{max-width:800px;margin:auto;padding:15px;}
    section{margin-bottom:20px;background:#1e1e1e;padding:15px;border-radius:8px;}
    input,textarea,select,button{width:100%;padding:8px;margin-top:5px;border-radius:5px;border:none;}
    button{background:#4caf50;color:white;cursor:pointer;}
    button:hover{background:#45a049;}
    .post{background:#2a2a2a;padding:10px;border-radius:6px;margin-bottom:10px;}
    .likes-comments{display:flex;gap:12px;margin-top:8px;cursor:pointer;}
    .comment-section input{width:80%;display:inline-block;}
    .comment-section button{width:18%;display:inline-block;}
    .admin-section{display:none;background:#1e1e1e;padding:15px;border-radius:8px;}
    .admin-section.show{display:block;}
    .message{color:#ff4d4d;margin-top:5px;font-size:0.9em;}
    img{max-width:100%;border-radius:6px;margin-top:8px;}
  </style>
</head>
<body>

<header>OpenSphere Nexus</header>

<main>
  <section id="loginSection">
    <h2>Login / Sign Up</h2>
    <input type="email" id="email" placeholder="Email" required>
    <input type="password" id="password" placeholder="Password" required>
    <label><input type="checkbox" id="acceptRules"> I accept the rules</label>
    <button id="loginBtn">Login / Sign Up</button>
    <div id="loginMessage" class="message"></div>
  </section>

  <section id="newPostSection">
    <h2>New Post</h2>
    <textarea id="postText" placeholder="What's happening?" rows="3"></textarea>
    <input type="text" id="postImage" placeholder="Image URL (optional)">
    <select id="postTag">
      <option value="General">General</option>
      <option value="Music">Music</option>
      <option value="Tech">Tech</option>
      <option value="Events">Events</option>
    </select>
    <button id="postBtn">Post</button>
    <div id="postMessage" class="message"></div>
  </section>

  <section id="feedSection">
    <h2>Feed</h2>
    <select id="filterTag">
      <option value="All">All</option>
      <option value="General">General</option>
      <option value="Music">Music</option>
      <option value="Tech">Tech</option>
      <option value="Events">Events</option>
    </select>
    <div id="feed"></div>
  </section>

  <section class="admin-section" id="adminPanel">
    <h2>Admin Panel</h2>
    <div id="adminPosts"></div>
  </section>
</main>

<script type="module">
  import { initializeApp } from "https://www.gstatic.com/firebasejs/10.12.2/firebase-app.js";
  import { getFirestore, collection, addDoc, onSnapshot, query, orderBy, serverTimestamp, doc, updateDoc, deleteDoc, getDoc } from "https://www.gstatic.com/firebasejs/10.12.2/firebase-firestore.js";
  import { getAuth, signInWithEmailAndPassword, createUserWithEmailAndPassword, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/10.12.2/firebase-auth.js";

  const firebaseConfig = {
    apiKey: "AIzaSyDquBAv_q3w2aJV_vmrrrNunUOnnZ2mLYc",
    authDomain: "opensphere-4bb89.firebaseapp.com",
    projectId: "opensphere-4bb89",
    storageBucket: "opensphere-4bb89.appspot.com",
    messagingSenderId: "374472445556",
    appId: "1:374472445556:web:64c4dd2c1984c81ec68b21"
  };

  const app = initializeApp(firebaseConfig);
  const db = getFirestore(app);
  const auth = getAuth(app);

  let currentUser = "";
  let adminEmail = "your@email.com"; // CHANGE THIS

  document.getElementById("loginBtn").addEventListener("click", async () => {
    const email = document.getElementById("email").value.trim();
    const password = document.getElementById("password").value.trim();
    const accept = document.getElementById("acceptRules").checked;
    const msg = document.getElementById("loginMessage");

    if (!accept) return msg.textContent = "Accept rules first.";
    if (!email || !password) return msg.textContent = "Fill both fields.";

    try {
      await signInWithEmailAndPassword(auth, email, password);
    } catch {
      try {
        await createUserWithEmailAndPassword(auth, email, password);
      } catch (e) {
        msg.textContent = e.message;
        return;
      }
    }
    msg.textContent = "Logged in!";
    msg.style.color = "green";
  });

  onAuthStateChanged(auth, (user) => {
    currentUser = user ? user.email : "";
    if (currentUser === adminEmail) document.getElementById("adminPanel").classList.add("show");
  });

  document.getElementById("postBtn").addEventListener("click", async () => {
    const text = document.getElementById("postText").value.trim();
    const image = document.getElementById("postImage").value.trim();
    const tag = document.getElementById("postTag").value;

    if (!text || !currentUser) return alert("Login first!");
    await addDoc(collection(db, "posts"), {
      text,
      image,
      tag,
      user: currentUser,
      username: currentUser.split("@")[0],
      timestamp: serverTimestamp(),
      likes: 0,
      comments: [],
      flagged: false
    });
    document.getElementById("postText").value = "";
    document.getElementById("postImage").value = "";
  });

  document.getElementById("filterTag").addEventListener("change", renderFeed);
  const postsQuery = query(collection(db, "posts"), orderBy("timestamp", "desc"));
  onSnapshot(postsQuery, snapshot => renderFeed(snapshot));

  function renderFeed(snapshot) {
    const feedDiv = document.getElementById("feed");
    const adminDiv = document.getElementById("adminPosts");
    feedDiv.innerHTML = "";
    adminDiv.innerHTML = "";

    const filter = document.getElementById("filterTag").value;

    snapshot.forEach(docSnap => {
      const data = docSnap.data();
      if (filter !== "All" && data.tag !== filter) return;

      const postDiv = document.createElement("div");
      postDiv.className = "post";

      let flaggedNotice = "";
      if (data.flagged) flaggedNotice = (currentUser === adminEmail) ? " " : "⚠️";

      postDiv.innerHTML = `
        <div>${data.text}</div>
        ${data.image ? `<img src="${data.image}" alt="Post Image"/>` : ""}
        <div class="likes-comments">❤️ ${data.likes || 0} | Comments: ${data.comments?.length || 0}</div>
        ${flaggedNotice}
        <div class="comment-section">
          <input type="text" id="comment-${docSnap.id}" placeholder="Add comment">
          <button onclick="addComment('${docSnap.id}')">Post</button>
        </div>
      `;
      feedDiv.appendChild(postDiv);

      if (data.flagged && currentUser === adminEmail) {
        const aDiv = document.createElement("div");
        aDiv.className = "post";
        aDiv.innerHTML = `<p>${data.user}: ${data.text}</p>
          <button onclick="approvePost('${docSnap.id}')">Approve</button>
          <button onclick="removePost('${docSnap.id}')">Remove</button>`;
        adminDiv.appendChild(aDiv);
      }
    });
  }

  window.addComment = async (postId) => {
    const input = document.getElementById(`comment-${postId}`);
    const text = input.value.trim();
    if (!text) return;
    const postRef = doc(db, "posts", postId);
    const postSnap = await getDoc(postRef);
    const comments = postSnap.exists() ? [...(postSnap.data().comments || []), {text}] : ;
    await updateDoc(postRef, { comments });
    input.value = "";
  };

  window.approvePost = async (id) => await updateDoc(doc(db, "posts", id), { flagged: false });
  window.removePost = async (id) => await deleteDoc(doc(db, "posts", id));
</script>
</body>
</html>
