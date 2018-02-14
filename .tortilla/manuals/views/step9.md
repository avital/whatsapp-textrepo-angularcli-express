# Step 9: Mutations

[//]: # (head-end)


In addition to fetching data using queries, Apollo also helps you handle GraphQL mutations. In GraphQL, mutations are identical to queries in syntax, the only difference being that you use the keyword mutation instead of query to indicate that the root fields on this query are going to be performing writes to the backend.
GraphQL mutations represent two things in one query string:

1. The mutation field name with arguments, which represents the actual operation to be done on the server.
2. The fields you want back from the result of the mutation to update the client.

When we use mutations in Apollo, the result is typically integrated into the cache automatically based on the id of the result, which in turn updates the UI automatically, so we often don't need to explicitly handle the results. In order for the client to correctly do this, we need to ensure we select the necessary fields in the result. One good strategy can be to simply ask for any fields that might have been affected by the mutation. Alternatively, you can use fragments to share the fields between a query and a mutation that updates that query.

## Server

Finally we're going to create our mutations in the server:

[{]: <helper> (diffStep "3.1" module="server")

#### [Step 3.1: Add mutations](https://github.com/Urigo/whatsapp-textrepo-angularcli-express/commit/59d0440)

##### Changed schema&#x2F;resolvers.ts
```diff
@@ -1,5 +1,6 @@
-┊1┊ ┊import { db } from "../db";
-┊2┊ ┊import { IResolvers } from '../types';
+┊ ┊1┊import { IResolvers } from "../types";
+┊ ┊2┊import { ChatDb, db, MessageDb, MessageType, RecipientDb } from "../db";
+┊ ┊3┊import moment from "moment";
 ┊3┊4┊
 ┊4┊5┊let users = db.users;
 ┊5┊6┊let chats = db.chats;
```
```diff
@@ -12,8 +13,278 @@
 ┊ 12┊ 13┊    chats: () => chats.filter(chat => chat.listingMemberIds.includes(currentUser)),
 ┊ 13┊ 14┊    chat: (obj, {chatId}) => chats.find(chat => chat.id === Number(chatId)),
 ┊ 14┊ 15┊  },
+┊   ┊ 16┊  Mutation: {
+┊   ┊ 17┊    addChat: (obj, {recipientId}) => {
+┊   ┊ 18┊      if (!users.find(user => user.id === Number(recipientId))) {
+┊   ┊ 19┊        throw new Error(`Recipient ${recipientId} doesn't exist.`);
+┊   ┊ 20┊      }
+┊   ┊ 21┊
+┊   ┊ 22┊      const chat = chats.find(chat => !chat.name && chat.allTimeMemberIds.includes(currentUser) && chat.allTimeMemberIds.includes(Number(recipientId)));
+┊   ┊ 23┊      if (chat) {
+┊   ┊ 24┊        // Chat already exists. Both users are already in the allTimeMemberIds array
+┊   ┊ 25┊        const chatId = chat.id;
+┊   ┊ 26┊        if (!chat.listingMemberIds.includes(currentUser)) {
+┊   ┊ 27┊          // The chat isn't listed for the current user. Add him to the memberIds
+┊   ┊ 28┊          chat.listingMemberIds.push(currentUser);
+┊   ┊ 29┊          chats.find(chat => chat.id === chatId)!.listingMemberIds.push(currentUser);
+┊   ┊ 30┊          return chat;
+┊   ┊ 31┊        } else {
+┊   ┊ 32┊          throw new Error(`Chat already exists.`);
+┊   ┊ 33┊        }
+┊   ┊ 34┊      } else {
+┊   ┊ 35┊        // Create the chat
+┊   ┊ 36┊        const id = (chats.length && chats[chats.length - 1].id + 1) || 1;
+┊   ┊ 37┊        const chat: ChatDb = {
+┊   ┊ 38┊          id,
+┊   ┊ 39┊          name: null,
+┊   ┊ 40┊          picture: null,
+┊   ┊ 41┊          adminIds: null,
+┊   ┊ 42┊          ownerId: null,
+┊   ┊ 43┊          allTimeMemberIds: [currentUser, Number(recipientId)],
+┊   ┊ 44┊          // Chat will not be listed to the other user until the first message gets written
+┊   ┊ 45┊          listingMemberIds: [currentUser],
+┊   ┊ 46┊          actualGroupMemberIds: null,
+┊   ┊ 47┊          messages: [],
+┊   ┊ 48┊        };
+┊   ┊ 49┊        chats.push(chat);
+┊   ┊ 50┊        return chat;
+┊   ┊ 51┊      }
+┊   ┊ 52┊    },
+┊   ┊ 53┊    addGroup: (obj, {recipientIds, groupName}) => {
+┊   ┊ 54┊      recipientIds.forEach(recipientId => {
+┊   ┊ 55┊        if (!users.find(user => user.id === Number(recipientId))) {
+┊   ┊ 56┊          throw new Error(`Recipient ${recipientId} doesn't exist.`);
+┊   ┊ 57┊        }
+┊   ┊ 58┊      });
+┊   ┊ 59┊
+┊   ┊ 60┊      const id = (chats.length && chats[chats.length - 1].id + 1) || 1;
+┊   ┊ 61┊      const chat: ChatDb = {
+┊   ┊ 62┊        id,
+┊   ┊ 63┊        name: groupName,
+┊   ┊ 64┊        picture: null,
+┊   ┊ 65┊        adminIds: [currentUser],
+┊   ┊ 66┊        ownerId: currentUser,
+┊   ┊ 67┊        allTimeMemberIds: [currentUser, ...recipientIds.map(id => Number(id))],
+┊   ┊ 68┊        listingMemberIds: [currentUser, ...recipientIds.map(id => Number(id))],
+┊   ┊ 69┊        actualGroupMemberIds: [currentUser, ...recipientIds.map(id => Number(id))],
+┊   ┊ 70┊        messages: [],
+┊   ┊ 71┊      };
+┊   ┊ 72┊      chats.push(chat);
+┊   ┊ 73┊      return chat;
+┊   ┊ 74┊    },
+┊   ┊ 75┊    removeChat: (obj, {chatId}) => {
+┊   ┊ 76┊      const chat = chats.find(chat => chat.id === Number(chatId));
+┊   ┊ 77┊
+┊   ┊ 78┊      if (!chat) {
+┊   ┊ 79┊        throw new Error(`The chat ${chatId} doesn't exist.`);
+┊   ┊ 80┊      }
+┊   ┊ 81┊
+┊   ┊ 82┊      if (!chat.name) {
+┊   ┊ 83┊        // Chat
+┊   ┊ 84┊        if (!chat.listingMemberIds.includes(currentUser)) {
+┊   ┊ 85┊          throw new Error(`The user is not a member of the chat ${chatId}.`);
+┊   ┊ 86┊        }
+┊   ┊ 87┊
+┊   ┊ 88┊        // Instead of chaining map and filter we can loop once using reduce
+┊   ┊ 89┊        const messages = chat.messages.reduce<MessageDb[]>((filtered, message) => {
+┊   ┊ 90┊          // Remove the current user from the message holders
+┊   ┊ 91┊          message.holderIds = message.holderIds.filter(holderId => holderId !== currentUser);
+┊   ┊ 92┊
+┊   ┊ 93┊          if (message.holderIds.length !== 0) {
+┊   ┊ 94┊            filtered.push(message);
+┊   ┊ 95┊          } // else discard the message
+┊   ┊ 96┊
+┊   ┊ 97┊          return filtered;
+┊   ┊ 98┊        }, []);
+┊   ┊ 99┊
+┊   ┊100┊        // Remove the current user from who gets the chat listed. The chat will no longer appear in his list
+┊   ┊101┊        const listingMemberIds = chat.listingMemberIds.filter(listingId => listingId !== currentUser);
+┊   ┊102┊
+┊   ┊103┊        // Check how many members are left
+┊   ┊104┊        if (listingMemberIds.length === 0) {
+┊   ┊105┊          // Delete the chat
+┊   ┊106┊          chats = chats.filter(chat => chat.id !== Number(chatId));
+┊   ┊107┊        } else {
+┊   ┊108┊          // Update the chat
+┊   ┊109┊          chats = chats.map(chat => {
+┊   ┊110┊            if (chat.id === Number(chatId)) {
+┊   ┊111┊              chat = {...chat, listingMemberIds, messages};
+┊   ┊112┊            }
+┊   ┊113┊            return chat;
+┊   ┊114┊          });
+┊   ┊115┊        }
+┊   ┊116┊        return chatId;
+┊   ┊117┊      } else {
+┊   ┊118┊        // Group
+┊   ┊119┊        if (chat.ownerId !== currentUser) {
+┊   ┊120┊          throw new Error(`Group ${chatId} is not owned by the user.`);
+┊   ┊121┊        }
+┊   ┊122┊
+┊   ┊123┊        // Instead of chaining map and filter we can loop once using reduce
+┊   ┊124┊        const messages = chat.messages.reduce<MessageDb[]>((filtered, message) => {
+┊   ┊125┊          // Remove the current user from the message holders
+┊   ┊126┊          message.holderIds = message.holderIds.filter(holderId => holderId !== currentUser);
+┊   ┊127┊
+┊   ┊128┊          if (message.holderIds.length !== 0) {
+┊   ┊129┊            filtered.push(message);
+┊   ┊130┊          } // else discard the message
+┊   ┊131┊
+┊   ┊132┊          return filtered;
+┊   ┊133┊        }, []);
+┊   ┊134┊
+┊   ┊135┊        // Remove the current user from who gets the group listed. The group will no longer appear in his list
+┊   ┊136┊        const listingMemberIds = chat.listingMemberIds.filter(listingId => listingId !== currentUser);
+┊   ┊137┊
+┊   ┊138┊        // Check how many members (including previous ones who can still access old messages) are left
+┊   ┊139┊        if (listingMemberIds.length === 0) {
+┊   ┊140┊          // Remove the group
+┊   ┊141┊          chats = chats.filter(chat => chat.id !== Number(chatId));
+┊   ┊142┊        } else {
+┊   ┊143┊          // Update the group
+┊   ┊144┊
+┊   ┊145┊          // Remove the current user from the chat members. He is no longer a member of the group
+┊   ┊146┊          const actualGroupMemberIds = chat.actualGroupMemberIds!.filter(memberId => memberId !== currentUser);
+┊   ┊147┊          // Remove the current user from the chat admins
+┊   ┊148┊          const adminIds = chat.adminIds!.filter(memberId => memberId !== currentUser);
+┊   ┊149┊          // Set the owner id to be null. A null owner means the group is read-only
+┊   ┊150┊          let ownerId: number | null = null;
+┊   ┊151┊
+┊   ┊152┊          // Check if there is any admin left
+┊   ┊153┊          if (adminIds!.length) {
+┊   ┊154┊            // Pick an admin as the new owner. The group is no longer read-only
+┊   ┊155┊            ownerId = chat.adminIds![0];
+┊   ┊156┊          }
+┊   ┊157┊
+┊   ┊158┊          chats = chats.map(chat => {
+┊   ┊159┊            if (chat.id === Number(chatId)) {
+┊   ┊160┊              chat = {...chat, messages, listingMemberIds, actualGroupMemberIds, adminIds, ownerId};
+┊   ┊161┊            }
+┊   ┊162┊            return chat;
+┊   ┊163┊          });
+┊   ┊164┊        }
+┊   ┊165┊        return chatId;
+┊   ┊166┊      }
+┊   ┊167┊    },
+┊   ┊168┊    addMessage: (obj, {chatId, content}) => {
+┊   ┊169┊      if (content === null || content === '') {
+┊   ┊170┊        throw new Error(`Cannot add empty or null messages.`);
+┊   ┊171┊      }
+┊   ┊172┊
+┊   ┊173┊      let chat = chats.find(chat => chat.id === Number(chatId));
+┊   ┊174┊
+┊   ┊175┊      if (!chat) {
+┊   ┊176┊        throw new Error(`Cannot find chat ${chatId}.`);
+┊   ┊177┊      }
+┊   ┊178┊
+┊   ┊179┊      let holderIds = chat.listingMemberIds;
+┊   ┊180┊
+┊   ┊181┊      if (!chat.name) {
+┊   ┊182┊        // Chat
+┊   ┊183┊        if (!chat.listingMemberIds.find(listingId => listingId === currentUser)) {
+┊   ┊184┊          throw new Error(`The chat ${chatId} must be listed for the current user before adding a message.`);
+┊   ┊185┊        }
+┊   ┊186┊
+┊   ┊187┊        const recipientId = chat.allTimeMemberIds.filter(userId => userId !== currentUser)[0];
+┊   ┊188┊
+┊   ┊189┊        if (!chat.listingMemberIds.find(listingId => listingId === recipientId)) {
+┊   ┊190┊          // Chat is not listed for the recipient. Add him to the listingMemberIds
+┊   ┊191┊          const listingMemberIds = chat.listingMemberIds.concat(recipientId);
+┊   ┊192┊
+┊   ┊193┊          chats = chats.map(chat => {
+┊   ┊194┊            if (chat.id === Number(chatId)) {
+┊   ┊195┊              chat = {...chat, listingMemberIds};
+┊   ┊196┊            }
+┊   ┊197┊            return chat;
+┊   ┊198┊          });
+┊   ┊199┊
+┊   ┊200┊          holderIds = listingMemberIds;
+┊   ┊201┊        }
+┊   ┊202┊      } else {
+┊   ┊203┊        // Group
+┊   ┊204┊        if (!chat.actualGroupMemberIds!.find(memberId => memberId === currentUser)) {
+┊   ┊205┊          throw new Error(`The user is not a member of the group ${chatId}. Cannot add message.`);
+┊   ┊206┊        }
+┊   ┊207┊
+┊   ┊208┊        holderIds = chat.actualGroupMemberIds!;
+┊   ┊209┊      }
+┊   ┊210┊
+┊   ┊211┊      const id = (chat.messages.length && chat.messages[chat.messages.length - 1].id + 1) || 1;
+┊   ┊212┊
+┊   ┊213┊      let recipients: RecipientDb[] = [];
+┊   ┊214┊
+┊   ┊215┊      holderIds.forEach(holderId => {
+┊   ┊216┊        if (holderId !== currentUser) {
+┊   ┊217┊          recipients.push({
+┊   ┊218┊            userId: holderId,
+┊   ┊219┊            messageId: id,
+┊   ┊220┊            chatId: Number(chatId),
+┊   ┊221┊            receivedAt: null,
+┊   ┊222┊            readAt: null,
+┊   ┊223┊          });
+┊   ┊224┊        }
+┊   ┊225┊      });
+┊   ┊226┊
+┊   ┊227┊      const message: MessageDb = {
+┊   ┊228┊        id,
+┊   ┊229┊        chatId: Number(chatId),
+┊   ┊230┊        senderId: currentUser,
+┊   ┊231┊        content,
+┊   ┊232┊        createdAt: moment().unix(),
+┊   ┊233┊        type: MessageType.TEXT,
+┊   ┊234┊        recipients,
+┊   ┊235┊        holderIds,
+┊   ┊236┊      };
+┊   ┊237┊
+┊   ┊238┊      chats = chats.map(chat => {
+┊   ┊239┊        if (chat.id === Number(chatId)) {
+┊   ┊240┊          chat = {...chat, messages: chat.messages.concat(message)}
+┊   ┊241┊        }
+┊   ┊242┊        return chat;
+┊   ┊243┊      });
+┊   ┊244┊
+┊   ┊245┊      return message;
+┊   ┊246┊    },
+┊   ┊247┊    removeMessages: (obj, {chatId, messageIds, all}) => {
+┊   ┊248┊      const chat = chats.find(chat => chat.id === Number(chatId));
+┊   ┊249┊
+┊   ┊250┊      if (!chat) {
+┊   ┊251┊        throw new Error(`Cannot find chat ${chatId}.`);
+┊   ┊252┊      }
+┊   ┊253┊
+┊   ┊254┊      if (!chat.listingMemberIds.find(listingId => listingId === currentUser)) {
+┊   ┊255┊        throw new Error(`The chat/group ${chatId} is not listed for the current user, so there is nothing to delete.`);
+┊   ┊256┊      }
+┊   ┊257┊
+┊   ┊258┊      if (all && messageIds) {
+┊   ┊259┊        throw new Error(`Cannot specify both 'all' and 'messageIds'.`);
+┊   ┊260┊      }
+┊   ┊261┊
+┊   ┊262┊      let deletedIds: string[] = [];
+┊   ┊263┊      chats = chats.map(chat => {
+┊   ┊264┊        if (chat.id === Number(chatId)) {
+┊   ┊265┊          // Instead of chaining map and filter we can loop once using reduce
+┊   ┊266┊          const messages = chat.messages.reduce<MessageDb[]>((filtered, message) => {
+┊   ┊267┊            if (all || messageIds!.map(Number).includes(message.id)) {
+┊   ┊268┊              deletedIds.push(String(message.id));
+┊   ┊269┊              // Remove the current user from the message holders
+┊   ┊270┊              message.holderIds = message.holderIds.filter(holderId => holderId !== currentUser);
+┊   ┊271┊            }
+┊   ┊272┊
+┊   ┊273┊            if (message.holderIds.length !== 0) {
+┊   ┊274┊              filtered.push(message);
+┊   ┊275┊            } // else discard the message
+┊   ┊276┊
+┊   ┊277┊            return filtered;
+┊   ┊278┊          }, []);
+┊   ┊279┊          chat = {...chat, messages};
+┊   ┊280┊        }
+┊   ┊281┊        return chat;
+┊   ┊282┊      });
+┊   ┊283┊      return deletedIds;
+┊   ┊284┊    },
+┊   ┊285┊  },
 ┊ 15┊286┊  Chat: {
-┊ 16┊   ┊    name: (chat): string => chat.name ? chat.name : users
+┊   ┊287┊    name: (chat) => chat.name ? chat.name : users
 ┊ 17┊288┊      .find(user => user.id === chat.allTimeMemberIds.find(userId => userId !== currentUser))!.name,
 ┊ 18┊289┊    picture: (chat) => chat.name ? chat.picture : users
 ┊ 19┊290┊      .find(user => user.id === chat.allTimeMemberIds.find(userId => userId !== currentUser))!.picture,
```

##### Changed schema&#x2F;typeDefs.ts
```diff
@@ -65,4 +65,20 @@
 ┊65┊65┊    picture: String
 ┊66┊66┊    phone: String
 ┊67┊67┊  }
+┊  ┊68┊
+┊  ┊69┊  type Mutation {
+┊  ┊70┊    addChat(recipientId: ID!): Chat
+┊  ┊71┊    addGroup(recipientIds: [ID!]!, groupName: String!): Chat
+┊  ┊72┊    removeChat(chatId: ID!): ID
+┊  ┊73┊    addMessage(chatId: ID!, content: String!): Message
+┊  ┊74┊    removeMessages(chatId: ID!, messageIds: [ID], all: Boolean): [ID]
+┊  ┊75┊    addMembers(groupId: ID!, userIds: [ID!]!): [ID]
+┊  ┊76┊    removeMembers(groupId: ID!, userIds: [ID!]!): [ID]
+┊  ┊77┊    addAdmins(groupId: ID!, userIds: [ID!]!): [ID]
+┊  ┊78┊    removeAdmins(groupId: ID!, userIds: [ID!]!): [ID]
+┊  ┊79┊    setGroupName(groupId: ID!): String
+┊  ┊80┊    setGroupPicture(groupId: ID!): String
+┊  ┊81┊    markAsReceived(chatId: ID!): Boolean
+┊  ┊82┊    markAsRead(chatId: ID!): Boolean
+┊  ┊83┊  }
 ┊68┊84┊`;
```

##### Changed types.d.ts
```diff
@@ -84,6 +84,34 @@
 ┊ 84┊ 84┊  readAt?: Maybe<string>;
 ┊ 85┊ 85┊}
 ┊ 86┊ 86┊
+┊   ┊ 87┊export interface Mutation {
+┊   ┊ 88┊  addChat?: Maybe<Chat>;
+┊   ┊ 89┊
+┊   ┊ 90┊  addGroup?: Maybe<Chat>;
+┊   ┊ 91┊
+┊   ┊ 92┊  removeChat?: Maybe<string>;
+┊   ┊ 93┊
+┊   ┊ 94┊  addMessage?: Maybe<Message>;
+┊   ┊ 95┊
+┊   ┊ 96┊  removeMessages?: Maybe<(Maybe<string>)[]>;
+┊   ┊ 97┊
+┊   ┊ 98┊  addMembers?: Maybe<(Maybe<string>)[]>;
+┊   ┊ 99┊
+┊   ┊100┊  removeMembers?: Maybe<(Maybe<string>)[]>;
+┊   ┊101┊
+┊   ┊102┊  addAdmins?: Maybe<(Maybe<string>)[]>;
+┊   ┊103┊
+┊   ┊104┊  removeAdmins?: Maybe<(Maybe<string>)[]>;
+┊   ┊105┊
+┊   ┊106┊  setGroupName?: Maybe<string>;
+┊   ┊107┊
+┊   ┊108┊  setGroupPicture?: Maybe<string>;
+┊   ┊109┊
+┊   ┊110┊  markAsReceived?: Maybe<boolean>;
+┊   ┊111┊
+┊   ┊112┊  markAsRead?: Maybe<boolean>;
+┊   ┊113┊}
+┊   ┊114┊
 ┊ 87┊115┊// ====================================================
 ┊ 88┊116┊// Arguments
 ┊ 89┊117┊// ====================================================
```
```diff
@@ -94,6 +122,61 @@
 ┊ 94┊122┊export interface MessagesChatArgs {
 ┊ 95┊123┊  amount?: Maybe<number>;
 ┊ 96┊124┊}
+┊   ┊125┊export interface AddChatMutationArgs {
+┊   ┊126┊  recipientId: string;
+┊   ┊127┊}
+┊   ┊128┊export interface AddGroupMutationArgs {
+┊   ┊129┊  recipientIds: string[];
+┊   ┊130┊
+┊   ┊131┊  groupName: string;
+┊   ┊132┊}
+┊   ┊133┊export interface RemoveChatMutationArgs {
+┊   ┊134┊  chatId: string;
+┊   ┊135┊}
+┊   ┊136┊export interface AddMessageMutationArgs {
+┊   ┊137┊  chatId: string;
+┊   ┊138┊
+┊   ┊139┊  content: string;
+┊   ┊140┊}
+┊   ┊141┊export interface RemoveMessagesMutationArgs {
+┊   ┊142┊  chatId: string;
+┊   ┊143┊
+┊   ┊144┊  messageIds?: Maybe<(Maybe<string>)[]>;
+┊   ┊145┊
+┊   ┊146┊  all?: Maybe<boolean>;
+┊   ┊147┊}
+┊   ┊148┊export interface AddMembersMutationArgs {
+┊   ┊149┊  groupId: string;
+┊   ┊150┊
+┊   ┊151┊  userIds: string[];
+┊   ┊152┊}
+┊   ┊153┊export interface RemoveMembersMutationArgs {
+┊   ┊154┊  groupId: string;
+┊   ┊155┊
+┊   ┊156┊  userIds: string[];
+┊   ┊157┊}
+┊   ┊158┊export interface AddAdminsMutationArgs {
+┊   ┊159┊  groupId: string;
+┊   ┊160┊
+┊   ┊161┊  userIds: string[];
+┊   ┊162┊}
+┊   ┊163┊export interface RemoveAdminsMutationArgs {
+┊   ┊164┊  groupId: string;
+┊   ┊165┊
+┊   ┊166┊  userIds: string[];
+┊   ┊167┊}
+┊   ┊168┊export interface SetGroupNameMutationArgs {
+┊   ┊169┊  groupId: string;
+┊   ┊170┊}
+┊   ┊171┊export interface SetGroupPictureMutationArgs {
+┊   ┊172┊  groupId: string;
+┊   ┊173┊}
+┊   ┊174┊export interface MarkAsReceivedMutationArgs {
+┊   ┊175┊  chatId: string;
+┊   ┊176┊}
+┊   ┊177┊export interface MarkAsReadMutationArgs {
+┊   ┊178┊  chatId: string;
+┊   ┊179┊}
 ┊ 97┊180┊
 ┊ 98┊181┊import { GraphQLResolveInfo } from "graphql";
 ┊ 99┊182┊
```
```diff
@@ -408,6 +491,197 @@
 ┊408┊491┊  > = Resolver<R, Parent, Context>;
 ┊409┊492┊}
 ┊410┊493┊
+┊   ┊494┊export namespace MutationResolvers {
+┊   ┊495┊  export interface Resolvers<Context = {}, TypeParent = {}> {
+┊   ┊496┊    addChat?: AddChatResolver<Maybe<ChatDb>, TypeParent, Context>;
+┊   ┊497┊
+┊   ┊498┊    addGroup?: AddGroupResolver<Maybe<ChatDb>, TypeParent, Context>;
+┊   ┊499┊
+┊   ┊500┊    removeChat?: RemoveChatResolver<Maybe<string>, TypeParent, Context>;
+┊   ┊501┊
+┊   ┊502┊    addMessage?: AddMessageResolver<Maybe<MessageDb>, TypeParent, Context>;
+┊   ┊503┊
+┊   ┊504┊    removeMessages?: RemoveMessagesResolver<
+┊   ┊505┊      Maybe<(Maybe<string>)[]>,
+┊   ┊506┊      TypeParent,
+┊   ┊507┊      Context
+┊   ┊508┊    >;
+┊   ┊509┊
+┊   ┊510┊    addMembers?: AddMembersResolver<
+┊   ┊511┊      Maybe<(Maybe<string>)[]>,
+┊   ┊512┊      TypeParent,
+┊   ┊513┊      Context
+┊   ┊514┊    >;
+┊   ┊515┊
+┊   ┊516┊    removeMembers?: RemoveMembersResolver<
+┊   ┊517┊      Maybe<(Maybe<string>)[]>,
+┊   ┊518┊      TypeParent,
+┊   ┊519┊      Context
+┊   ┊520┊    >;
+┊   ┊521┊
+┊   ┊522┊    addAdmins?: AddAdminsResolver<
+┊   ┊523┊      Maybe<(Maybe<string>)[]>,
+┊   ┊524┊      TypeParent,
+┊   ┊525┊      Context
+┊   ┊526┊    >;
+┊   ┊527┊
+┊   ┊528┊    removeAdmins?: RemoveAdminsResolver<
+┊   ┊529┊      Maybe<(Maybe<string>)[]>,
+┊   ┊530┊      TypeParent,
+┊   ┊531┊      Context
+┊   ┊532┊    >;
+┊   ┊533┊
+┊   ┊534┊    setGroupName?: SetGroupNameResolver<Maybe<string>, TypeParent, Context>;
+┊   ┊535┊
+┊   ┊536┊    setGroupPicture?: SetGroupPictureResolver<
+┊   ┊537┊      Maybe<string>,
+┊   ┊538┊      TypeParent,
+┊   ┊539┊      Context
+┊   ┊540┊    >;
+┊   ┊541┊
+┊   ┊542┊    markAsReceived?: MarkAsReceivedResolver<
+┊   ┊543┊      Maybe<boolean>,
+┊   ┊544┊      TypeParent,
+┊   ┊545┊      Context
+┊   ┊546┊    >;
+┊   ┊547┊
+┊   ┊548┊    markAsRead?: MarkAsReadResolver<Maybe<boolean>, TypeParent, Context>;
+┊   ┊549┊  }
+┊   ┊550┊
+┊   ┊551┊  export type AddChatResolver<
+┊   ┊552┊    R = Maybe<ChatDb>,
+┊   ┊553┊    Parent = {},
+┊   ┊554┊    Context = {}
+┊   ┊555┊  > = Resolver<R, Parent, Context, AddChatArgs>;
+┊   ┊556┊  export interface AddChatArgs {
+┊   ┊557┊    recipientId: string;
+┊   ┊558┊  }
+┊   ┊559┊
+┊   ┊560┊  export type AddGroupResolver<
+┊   ┊561┊    R = Maybe<ChatDb>,
+┊   ┊562┊    Parent = {},
+┊   ┊563┊    Context = {}
+┊   ┊564┊  > = Resolver<R, Parent, Context, AddGroupArgs>;
+┊   ┊565┊  export interface AddGroupArgs {
+┊   ┊566┊    recipientIds: string[];
+┊   ┊567┊
+┊   ┊568┊    groupName: string;
+┊   ┊569┊  }
+┊   ┊570┊
+┊   ┊571┊  export type RemoveChatResolver<
+┊   ┊572┊    R = Maybe<string>,
+┊   ┊573┊    Parent = {},
+┊   ┊574┊    Context = {}
+┊   ┊575┊  > = Resolver<R, Parent, Context, RemoveChatArgs>;
+┊   ┊576┊  export interface RemoveChatArgs {
+┊   ┊577┊    chatId: string;
+┊   ┊578┊  }
+┊   ┊579┊
+┊   ┊580┊  export type AddMessageResolver<
+┊   ┊581┊    R = Maybe<MessageDb>,
+┊   ┊582┊    Parent = {},
+┊   ┊583┊    Context = {}
+┊   ┊584┊  > = Resolver<R, Parent, Context, AddMessageArgs>;
+┊   ┊585┊  export interface AddMessageArgs {
+┊   ┊586┊    chatId: string;
+┊   ┊587┊
+┊   ┊588┊    content: string;
+┊   ┊589┊  }
+┊   ┊590┊
+┊   ┊591┊  export type RemoveMessagesResolver<
+┊   ┊592┊    R = Maybe<(Maybe<string>)[]>,
+┊   ┊593┊    Parent = {},
+┊   ┊594┊    Context = {}
+┊   ┊595┊  > = Resolver<R, Parent, Context, RemoveMessagesArgs>;
+┊   ┊596┊  export interface RemoveMessagesArgs {
+┊   ┊597┊    chatId: string;
+┊   ┊598┊
+┊   ┊599┊    messageIds?: Maybe<(Maybe<string>)[]>;
+┊   ┊600┊
+┊   ┊601┊    all?: Maybe<boolean>;
+┊   ┊602┊  }
+┊   ┊603┊
+┊   ┊604┊  export type AddMembersResolver<
+┊   ┊605┊    R = Maybe<(Maybe<string>)[]>,
+┊   ┊606┊    Parent = {},
+┊   ┊607┊    Context = {}
+┊   ┊608┊  > = Resolver<R, Parent, Context, AddMembersArgs>;
+┊   ┊609┊  export interface AddMembersArgs {
+┊   ┊610┊    groupId: string;
+┊   ┊611┊
+┊   ┊612┊    userIds: string[];
+┊   ┊613┊  }
+┊   ┊614┊
+┊   ┊615┊  export type RemoveMembersResolver<
+┊   ┊616┊    R = Maybe<(Maybe<string>)[]>,
+┊   ┊617┊    Parent = {},
+┊   ┊618┊    Context = {}
+┊   ┊619┊  > = Resolver<R, Parent, Context, RemoveMembersArgs>;
+┊   ┊620┊  export interface RemoveMembersArgs {
+┊   ┊621┊    groupId: string;
+┊   ┊622┊
+┊   ┊623┊    userIds: string[];
+┊   ┊624┊  }
+┊   ┊625┊
+┊   ┊626┊  export type AddAdminsResolver<
+┊   ┊627┊    R = Maybe<(Maybe<string>)[]>,
+┊   ┊628┊    Parent = {},
+┊   ┊629┊    Context = {}
+┊   ┊630┊  > = Resolver<R, Parent, Context, AddAdminsArgs>;
+┊   ┊631┊  export interface AddAdminsArgs {
+┊   ┊632┊    groupId: string;
+┊   ┊633┊
+┊   ┊634┊    userIds: string[];
+┊   ┊635┊  }
+┊   ┊636┊
+┊   ┊637┊  export type RemoveAdminsResolver<
+┊   ┊638┊    R = Maybe<(Maybe<string>)[]>,
+┊   ┊639┊    Parent = {},
+┊   ┊640┊    Context = {}
+┊   ┊641┊  > = Resolver<R, Parent, Context, RemoveAdminsArgs>;
+┊   ┊642┊  export interface RemoveAdminsArgs {
+┊   ┊643┊    groupId: string;
+┊   ┊644┊
+┊   ┊645┊    userIds: string[];
+┊   ┊646┊  }
+┊   ┊647┊
+┊   ┊648┊  export type SetGroupNameResolver<
+┊   ┊649┊    R = Maybe<string>,
+┊   ┊650┊    Parent = {},
+┊   ┊651┊    Context = {}
+┊   ┊652┊  > = Resolver<R, Parent, Context, SetGroupNameArgs>;
+┊   ┊653┊  export interface SetGroupNameArgs {
+┊   ┊654┊    groupId: string;
+┊   ┊655┊  }
+┊   ┊656┊
+┊   ┊657┊  export type SetGroupPictureResolver<
+┊   ┊658┊    R = Maybe<string>,
+┊   ┊659┊    Parent = {},
+┊   ┊660┊    Context = {}
+┊   ┊661┊  > = Resolver<R, Parent, Context, SetGroupPictureArgs>;
+┊   ┊662┊  export interface SetGroupPictureArgs {
+┊   ┊663┊    groupId: string;
+┊   ┊664┊  }
+┊   ┊665┊
+┊   ┊666┊  export type MarkAsReceivedResolver<
+┊   ┊667┊    R = Maybe<boolean>,
+┊   ┊668┊    Parent = {},
+┊   ┊669┊    Context = {}
+┊   ┊670┊  > = Resolver<R, Parent, Context, MarkAsReceivedArgs>;
+┊   ┊671┊  export interface MarkAsReceivedArgs {
+┊   ┊672┊    chatId: string;
+┊   ┊673┊  }
+┊   ┊674┊
+┊   ┊675┊  export type MarkAsReadResolver<
+┊   ┊676┊    R = Maybe<boolean>,
+┊   ┊677┊    Parent = {},
+┊   ┊678┊    Context = {}
+┊   ┊679┊  > = Resolver<R, Parent, Context, MarkAsReadArgs>;
+┊   ┊680┊  export interface MarkAsReadArgs {
+┊   ┊681┊    chatId: string;
+┊   ┊682┊  }
+┊   ┊683┊}
+┊   ┊684┊
 ┊411┊685┊/** Directs the executor to skip this field or fragment when the `if` argument is true. */
 ┊412┊686┊export type SkipDirectiveResolver<Result> = DirectiveResolverFn<
 ┊413┊687┊  Result,
```
```diff
@@ -447,6 +721,7 @@
 ┊447┊721┊  Chat?: ChatResolvers.Resolvers;
 ┊448┊722┊  Message?: MessageResolvers.Resolvers;
 ┊449┊723┊  Recipient?: RecipientResolvers.Resolvers;
+┊   ┊724┊  Mutation?: MutationResolvers.Resolvers;
 ┊450┊725┊}
 ┊451┊726┊
 ┊452┊727┊export interface IDirectiveResolvers<Result> {
```

[}]: #

Let me briefly explain what's going on here.
For each chat/group we store the `allTimeMemberIds`, `listingMemberIds` and `actualGroupMemberIds` properties in our NoSQL-like fake db.
What's the difference between `allTimeMemberIds` and `listingMemberIds`? When a chat gets created only the user who created it will be able too see it, the chat will be displayed to the other user only once the first messaged gets sent. `allTimeMemberIds` is an array which always contain both the users, while `listingMemberIds` contains only the users which get the chat listed (initially the creator, later both users). `actualGroupMemberIds` is only used for groups.
Groups, instead, get listed by all members immediately since the creation. So initially both `allTimeMemberIds`, `listingMemberIds` and `actualGroupMemberIds` are similar. Later users can leave the group or get deleted (so they will be removed from `actualGroupMemberIds`) but they will still be able to list the group in read-only mode, thus remaining in the `listingMemberIds`. Once they remove the group they will also be remove from the `listingMemberIds` array.
That's why we have to check for several different conditions before adding/deleting messages: it could be necessary to add the other peer to the `listingMemberIds` (for example if we are writing the first message of a chat) or it could be necessary to physically remove the messages instead of simply removing the current user from the `holderIds`. `holderIds` is a field in each message which states which user will currently display that specific message. In fact each user can delete a specific message without affecting what the others will see. Once there will be no more users in the `holderIds` array it will be safe to delete the message.
Each message has also a `recipients` array containing the receiving date and the viewing date of that particular message for all the other users. That's necessary to implement the single, double and blue ticks used by the real Whatsapp.

It may seem a bit overwhelming at first, but you should keep in mind that the real Whatsapp has tons of features and also takes advantage of a local database to store messages, so it's easier for them to implement features like per-user messages: their source of truth is not the server because once downloaded the messages are kept in the client itself, so deleting messages doesn't affect anyone else. On the contrary our source of truth is the server, so our approach is more similar to Telegram instead. This is a better approach in my opinion because it allows us to show the messages for the same user on multiple clients, instead of having to rely on questionable approaches like Whatsapp Web.
Also we already implemented our mutations to take care of future use cases (like reading notifications) which we still didn't implement.

I said we were going to take greater advantage of `graphql-code-generator` once we started writing our first mutation and I'm going to show you why. Let's run the generator first:

    yarn generator

Then let's use the generated types:

[{]: <helper> (diffStep "3.3" module="server")

#### Step 3.3: NOT FOUND!

[}]: #

## Client

For the client I'll only show you how to make use of the addMessage mutation in this chapters. The other mutations will require much more boilerplate so I left them for their own chapter.

Let's start by wiring the addMessage mutation. We're going to write the GraphQL query and then use the generator to generate the types:

[{]: <helper> (diffStep "5.1" files="^\(?!src/types.d.ts$\).*" module="client")

#### [Step 5.1: Create addMessage mutation and generate types](https://github.com/Urigo/whatsapp-textrepo-angularcli-express/commit/4edff2f)

##### Changed src&#x2F;graphql.ts
```diff
@@ -10,6 +10,21 @@
 ┊10┊10┊// Documents
 ┊11┊11┊// ====================================================
 ┊12┊12┊
+┊  ┊13┊export namespace AddMessage {
+┊  ┊14┊  export type Variables = {
+┊  ┊15┊    chatId: string;
+┊  ┊16┊    content: string;
+┊  ┊17┊  };
+┊  ┊18┊
+┊  ┊19┊  export type Mutation = {
+┊  ┊20┊    __typename?: "Mutation";
+┊  ┊21┊
+┊  ┊22┊    addMessage: Maybe<AddMessage>;
+┊  ┊23┊  };
+┊  ┊24┊
+┊  ┊25┊  export type AddMessage = Message.Fragment;
+┊  ┊26┊}
+┊  ┊27┊
 ┊13┊28┊export namespace GetChat {
 ┊14┊29┊  export type Variables = {
 ┊15┊30┊    chatId: string;
```
```diff
@@ -229,6 +244,34 @@
 ┊229┊244┊  readAt?: Maybe<string>;
 ┊230┊245┊}
 ┊231┊246┊
+┊   ┊247┊export interface Mutation {
+┊   ┊248┊  addChat?: Maybe<Chat>;
+┊   ┊249┊
+┊   ┊250┊  addGroup?: Maybe<Chat>;
+┊   ┊251┊
+┊   ┊252┊  removeChat?: Maybe<string>;
+┊   ┊253┊
+┊   ┊254┊  addMessage?: Maybe<Message>;
+┊   ┊255┊
+┊   ┊256┊  removeMessages?: Maybe<(Maybe<string>)[]>;
+┊   ┊257┊
+┊   ┊258┊  addMembers?: Maybe<(Maybe<string>)[]>;
+┊   ┊259┊
+┊   ┊260┊  removeMembers?: Maybe<(Maybe<string>)[]>;
+┊   ┊261┊
+┊   ┊262┊  addAdmins?: Maybe<(Maybe<string>)[]>;
+┊   ┊263┊
+┊   ┊264┊  removeAdmins?: Maybe<(Maybe<string>)[]>;
+┊   ┊265┊
+┊   ┊266┊  setGroupName?: Maybe<string>;
+┊   ┊267┊
+┊   ┊268┊  setGroupPicture?: Maybe<string>;
+┊   ┊269┊
+┊   ┊270┊  markAsReceived?: Maybe<boolean>;
+┊   ┊271┊
+┊   ┊272┊  markAsRead?: Maybe<boolean>;
+┊   ┊273┊}
+┊   ┊274┊
 ┊232┊275┊// ====================================================
 ┊233┊276┊// Arguments
 ┊234┊277┊// ====================================================
```
```diff
@@ -239,6 +282,61 @@
 ┊239┊282┊export interface MessagesChatArgs {
 ┊240┊283┊  amount?: Maybe<number>;
 ┊241┊284┊}
+┊   ┊285┊export interface AddChatMutationArgs {
+┊   ┊286┊  recipientId: string;
+┊   ┊287┊}
+┊   ┊288┊export interface AddGroupMutationArgs {
+┊   ┊289┊  recipientIds: string[];
+┊   ┊290┊
+┊   ┊291┊  groupName: string;
+┊   ┊292┊}
+┊   ┊293┊export interface RemoveChatMutationArgs {
+┊   ┊294┊  chatId: string;
+┊   ┊295┊}
+┊   ┊296┊export interface AddMessageMutationArgs {
+┊   ┊297┊  chatId: string;
+┊   ┊298┊
+┊   ┊299┊  content: string;
+┊   ┊300┊}
+┊   ┊301┊export interface RemoveMessagesMutationArgs {
+┊   ┊302┊  chatId: string;
+┊   ┊303┊
+┊   ┊304┊  messageIds?: Maybe<(Maybe<string>)[]>;
+┊   ┊305┊
+┊   ┊306┊  all?: Maybe<boolean>;
+┊   ┊307┊}
+┊   ┊308┊export interface AddMembersMutationArgs {
+┊   ┊309┊  groupId: string;
+┊   ┊310┊
+┊   ┊311┊  userIds: string[];
+┊   ┊312┊}
+┊   ┊313┊export interface RemoveMembersMutationArgs {
+┊   ┊314┊  groupId: string;
+┊   ┊315┊
+┊   ┊316┊  userIds: string[];
+┊   ┊317┊}
+┊   ┊318┊export interface AddAdminsMutationArgs {
+┊   ┊319┊  groupId: string;
+┊   ┊320┊
+┊   ┊321┊  userIds: string[];
+┊   ┊322┊}
+┊   ┊323┊export interface RemoveAdminsMutationArgs {
+┊   ┊324┊  groupId: string;
+┊   ┊325┊
+┊   ┊326┊  userIds: string[];
+┊   ┊327┊}
+┊   ┊328┊export interface SetGroupNameMutationArgs {
+┊   ┊329┊  groupId: string;
+┊   ┊330┊}
+┊   ┊331┊export interface SetGroupPictureMutationArgs {
+┊   ┊332┊  groupId: string;
+┊   ┊333┊}
+┊   ┊334┊export interface MarkAsReceivedMutationArgs {
+┊   ┊335┊  chatId: string;
+┊   ┊336┊}
+┊   ┊337┊export interface MarkAsReadMutationArgs {
+┊   ┊338┊  chatId: string;
+┊   ┊339┊}
 ┊242┊340┊
 ┊243┊341┊// ====================================================
 ┊244┊342┊// START: Apollo Angular template
```
```diff
@@ -303,6 +401,23 @@
 ┊303┊401┊// Apollo Services
 ┊304┊402┊// ====================================================
 ┊305┊403┊
+┊   ┊404┊@Injectable({
+┊   ┊405┊  providedIn: "root"
+┊   ┊406┊})
+┊   ┊407┊export class AddMessageGQL extends Apollo.Mutation<
+┊   ┊408┊  AddMessage.Mutation,
+┊   ┊409┊  AddMessage.Variables
+┊   ┊410┊> {
+┊   ┊411┊  document: any = gql`
+┊   ┊412┊    mutation AddMessage($chatId: ID!, $content: String!) {
+┊   ┊413┊      addMessage(chatId: $chatId, content: $content) {
+┊   ┊414┊        ...Message
+┊   ┊415┊      }
+┊   ┊416┊    }
+┊   ┊417┊
+┊   ┊418┊    ${MessageFragment}
+┊   ┊419┊  `;
+┊   ┊420┊}
 ┊306┊421┊@Injectable({
 ┊307┊422┊  providedIn: "root"
 ┊308┊423┊})
```

##### Added src&#x2F;graphql&#x2F;addMessage.mutation.ts
```diff
@@ -0,0 +1,13 @@
+┊  ┊ 1┊import gql from 'graphql-tag';
+┊  ┊ 2┊import {fragments} from './fragment';
+┊  ┊ 3┊
+┊  ┊ 4┊// We use the gql tag to parse our query string into a query document
+┊  ┊ 5┊export const addMessageMutation = gql`
+┊  ┊ 6┊  mutation AddMessage($chatId: ID!, $content: String!) {
+┊  ┊ 7┊    addMessage(chatId: $chatId, content: $content) {
+┊  ┊ 8┊      ...Message
+┊  ┊ 9┊    }
+┊  ┊10┊  }
+┊  ┊11┊
+┊  ┊12┊  ${fragments['message']}
+┊  ┊13┊`;
```

[}]: #

Run the generator:

    yarn generator

Now let's use the just-created query:

[{]: <helper> (diffStep "5.2" module="client")

#### [Step 5.2: Modify chat component and service](https://github.com/Urigo/whatsapp-textrepo-angularcli-express/commit/3c53136)

##### Changed src&#x2F;app&#x2F;chat-viewer&#x2F;containers&#x2F;chat&#x2F;chat.component.ts
```diff
@@ -14,7 +14,7 @@
 ┊14┊14┊    </app-toolbar>
 ┊15┊15┊    <div class="container">
 ┊16┊16┊      <app-messages-list [items]="messages" [isGroup]="isGroup"></app-messages-list>
-┊17┊  ┊      <app-new-message></app-new-message>
+┊  ┊17┊      <app-new-message (newMessage)="addMessage($event)"></app-new-message>
 ┊18┊18┊    </div>
 ┊19┊19┊  `,
 ┊20┊20┊  styleUrls: ['./chat.component.scss']
```
```diff
@@ -44,4 +44,8 @@
 ┊44┊44┊  goToChats() {
 ┊45┊45┊    this.router.navigate(['/chats']);
 ┊46┊46┊  }
+┊  ┊47┊
+┊  ┊48┊  addMessage(content: string) {
+┊  ┊49┊    this.chatsService.addMessage(this.chatId, content).subscribe();
+┊  ┊50┊  }
 ┊47┊51┊}
```

##### Changed src&#x2F;app&#x2F;services&#x2F;chats.service.ts
```diff
@@ -1,6 +1,6 @@
 ┊1┊1┊import {map} from 'rxjs/operators';
 ┊2┊2┊import {Injectable} from '@angular/core';
-┊3┊ ┊import {GetChatsGQL, GetChatGQL} from '../../graphql';
+┊ ┊3┊import {GetChatsGQL, GetChatGQL, AddMessageGQL} from '../../graphql';
 ┊4┊4┊
 ┊5┊5┊@Injectable()
 ┊6┊6┊export class ChatsService {
```
```diff
@@ -8,7 +8,8 @@
 ┊ 8┊ 8┊
 ┊ 9┊ 9┊  constructor(
 ┊10┊10┊    private getChatsGQL: GetChatsGQL,
-┊11┊  ┊    private getChatGQL: GetChatGQL
+┊  ┊11┊    private getChatGQL: GetChatGQL,
+┊  ┊12┊    private addMessageGQL: AddMessageGQL
 ┊12┊13┊  ) {}
 ┊13┊14┊
 ┊14┊15┊  getChats() {
```
```diff
@@ -33,4 +34,11 @@
 ┊33┊34┊
 ┊34┊35┊    return {query, chat$};
 ┊35┊36┊  }
+┊  ┊37┊
+┊  ┊38┊  addMessage(chatId: string, content: string) {
+┊  ┊39┊    return this.addMessageGQL.mutate({
+┊  ┊40┊      chatId,
+┊  ┊41┊      content,
+┊  ┊42┊    });
+┊  ┊43┊  }
 ┊36┊44┊}
```

[}]: #

It's that simple! You would be tempted to say that it doesn't work, but you should try to refresh the page first ;)

[//]: # (foot-start)

[{]: <helper> (navStep)

| [< Previous Step](https://github.com/Urigo/whatsapp-textrepo-angularcli-express/tree/master@1.4.0/.tortilla/manuals/views/step8.md) | [Next Step >](https://github.com/Urigo/whatsapp-textrepo-angularcli-express/tree/master@1.4.0/.tortilla/manuals/views/step10.md) |
|:--------------------------------|--------------------------------:|

[}]: #