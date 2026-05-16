# Aurium-business-import { Client, Databases } from "node-appwrite"

export default async function main(req, res) {
  const client = new Client()
    .setEndpoint(process.env.APPWRITE_ENDPOINT) // ex: https://cloud.appwrite.io/v1
    .setProject(process.env.APPWRITE_PROJECT_ID)
    .setKey(process.env.APPWRITE_API_KEY)

  const databases = new Databases(client)

  try {
    // 1️⃣ Récupérer la transaction depuis l’événement
    const transaction = req.body // { user_id, amount, type }

    // 2️⃣ Charger l’utilisateur concerné
    const userDoc = await databases.getDocument(
      process.env.APPWRITE_DATABASE_ID,
      process.env.USERS_COLLECTION_ID,
      transaction.user_id
    )

    let newBalance = userDoc.balance

    // 3️⃣ Calculer le nouveau solde
    if (transaction.type === "deposit") {
      newBalance += transaction.amount
    } else if (transaction.type === "withdraw") {
      if (newBalance < transaction.amount) {
        throw new Error("Solde insuffisant")
      }
      newBalance -= transaction.amount
    }

    // 4️⃣ Mettre à jour le solde
    await databases.updateDocument(
      process.env.APPWRITE_DATABASE_ID,
      process.env.USERS_COLLECTION_ID,
      transaction.user_id,
      { balance: newBalance }
    )

    // 5️⃣ Créer une notification
    await databases.createDocument(
      process.env.APPWRITE_DATABASE_ID,
      process.env.NOTIFICATIONS_COLLECTION_ID,
      "unique()", // ID auto
      {
        user_id: transaction.user_id,
        message: `Transaction ${transaction.type} de ${transaction.amount} effectuée. Nouveau solde: ${newBalance}`
      }
    )

    res.json({ success: true, balance: newBalance })
  } catch (err) {
    console.error(err)
    res.json({ success: false, error: err.message })
  }
}