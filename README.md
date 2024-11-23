import Debug "mo:base/Debug";

// Kullanıcı profilini tutacak yapı
type Profile = {
    username: Text;
    bio: Text;
    followers: Set<Text>; // Takipçiler
    following: Set<Text>; // Takip edilenler
};

// Post yapısı
type Post = {
    postId: Nat;
    author: Text;
    content: Text;
    comments: [Comment];
    likes: Set<Text>;
};

// Yorum yapısı
type Comment = {
    commentId: Nat;
    author: Text;
    content: Text;
};

// Sosyal medya platformu
actor SocialMedia {
    var users: Map<Text, Profile> = Map.empty(); // Kullanıcı profilleri
    var posts: Map<Nat, Post> = Map.empty(); // Paylaşılan postlar
    var nextPostId: Nat = 0; // Sonraki post ID'si
    var nextCommentId: Nat = 0; // Sonraki yorum ID'si

    // Kullanıcı kaydı
    public func registerUser(username: Text, bio: Text) : async Text {
        if (Map.contains(users, username)) {
            return "Username already taken.";
        } else {
            let newProfile = Profile({
                username = username;
                bio = bio;
                followers = Set.empty();
                following = Set.empty();
            });
            users := Map.put(users, username, newProfile);
            return "User registered successfully!";
        }
    };

    // Kullanıcıyı takip etme
    public func followUser(follower: Text, following: Text) : async Text {
        switch (Map.get(users, follower)) {
            case (?followerProfile) {
                switch (Map.get(users, following)) {
                    case (?followingProfile) {
                        // Takip ilişkisini oluştur
                        users := Map.put(users, follower, Profile({
                            username = followerProfile.username;
                            bio = followerProfile.bio;
                            followers = followerProfile.followers;
                            following = Set.add(followerProfile.following, following);
                        }));
                        users := Map.put(users, following, Profile({
                            username = followingProfile.username;
                            bio = followingProfile.bio;
                            followers = Set.add(followingProfile.followers, follower);
                            following = followingProfile.following;
                        }));
                        return "Following user!";
                    };
                    case (_) {
                        return "User to follow not found!";
                    };
                }
            };
            case (_) {
                return "Follower not found!";
            };
        }
    };

    // Post paylaşma
    public func createPost(author: Text, content: Text) : async Text {
        if (!Map.contains(users, author)) {
            return "User not found!";
        } else {
            let newPost = Post({
                postId = nextPostId;
                author = author;
                content = content;
                comments = [];
                likes = Set.empty();
            });
            posts := Map.put(posts, nextPostId, newPost);
            nextPostId := nextPostId + 1;
            return "Post created successfully!";
        }
    };

    // Postları görüntüleme
    public func getPosts() : async [Post] {
        return Map.values(posts);
    };

    // Posta yorum yapma
    public func commentOnPost(postId: Nat, author: Text, content: Text) : async Text {
        if (!Map.contains(posts, postId)) {
            return "Post not found!";
        } else {
            let comment = Comment({
                commentId = nextCommentId;
                author = author;
                content = content;
            });
            let post = Map.get(posts, postId);
            switch (post) {
                case (?p) {
                    let updatedPost = Post({
                        postId = p.postId;
                        author = p.author;
                        content = p.content;
                        comments = p.comments # [comment];
                        likes = p.likes;
                    });
                    posts := Map.put(posts, postId, updatedPost);
                    nextCommentId := nextCommentId + 1;
                    return "Comment added!";
                };
                case (_) {
                    return "Error.";
                };
            }
        }
    };

    // Postu beğenme
    public func likePost(postId: Nat, user: Text) : async Text {
        if (!Map.contains(posts, postId)) {
            return "Post not found!";
        } else {
            let post = Map.get(posts, postId);
            switch (post) {
                case (?p) {
                    if (Set.contains(p.likes, user)) {
                        return "You already liked this post!";
                    } else {
                        let updatedPost = Post({
                            postId = p.postId;
                            author = p.author;
                            content = p.content;
                            comments = p.comments;
                            likes = Set.add(p.likes, user);
                        });
                        posts := Map.put(posts, postId, updatedPost);
                        return "Post liked!";
                    }
                };
                case (_) {
                    return "Error.";
                };
            }
        }
    };
};
